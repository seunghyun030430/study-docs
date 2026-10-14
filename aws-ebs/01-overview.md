# Amazon EBS 개요

> Amazon EBS(Elastic Block Store)는 EC2 인스턴스에 붙여 쓰는 **블록 스토리지(가상 디스크)**다. 인스턴스를 꺼도 데이터가 남고, **한 가용 영역(AZ) 안에서만** 연결할 수 있으며, **스냅샷**으로 백업해 다른 AZ·리전으로 옮긴다. 요금은 **만든 용량(GB-월)과 추가로 정한 성능**에 매겨지므로, 쓰지 않는 볼륨이나 오래된 스냅샷도 계속 과금된다.

이 문서는 EBS를 처음 보는 사람을 기준으로 용어를 한 단계씩 쌓아 간다. 처음 나오는 용어는 **굵게** 표시하고 바로 뜻을 적는다. 모든 용어는 맨 아래 [용어 정리](#용어-정리)에 모아 두었다.

## 1단계: 블록 스토리지란

컴퓨터의 하드디스크나 SSD를 떠올리면 된다. OS는 디스크를 일정한 크기의 칸(**블록**)으로 나눠 읽고 쓴다. 그 위에 **파일시스템**(XFS, ext4, NTFS 등)을 만들어야 파일을 저장할 수 있다.

**EBS 볼륨**은 이런 디스크를 AWS가 네트워크 너머에서 제공하는 것이다. 인스턴스 입장에서는 그냥 디스크 하나가 붙은 것처럼 보인다.

```
EC2 인스턴스
 ├─ /dev/nvme0n1  ← 루트 볼륨(EBS): OS가 설치된 디스크
 └─ /dev/nvme1n1  ← 데이터 볼륨(EBS): /data 에 마운트
```

> EBS는 "scalable, high-performance block storage"이며, 볼륨과 스냅샷 두 가지 리소스로 이루어진다. — [What is Amazon EBS?](https://docs.aws.amazon.com/ebs/latest/userguide/what-is-ebs.html)

## 2단계: EBS 볼륨의 성질

출처: [EBS volumes](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes.html), [What is Amazon EBS?](https://docs.aws.amazon.com/ebs/latest/userguide/what-is-ebs.html)

| 성질 | 내용 |
| --- | --- |
| **인스턴스와 독립** | "EBS volumes persist independently from the running life of an EC2 instance." 인스턴스를 멈추거나 다른 인스턴스로 옮겨 붙여도 데이터가 남음 |
| **AZ에 묶임** | **가용 영역(AZ)**은 한 리전 안의 독립된 데이터센터 묶음이다. 볼륨과 인스턴스는 **같은 AZ**여야 연결할 수 있음 |
| AZ 안 복제 | 데이터는 같은 AZ의 여러 서버에 자동 복제됨(다른 AZ로는 복제되지 않음) |
| **내구성** | io2: 99.999%(연간 고장률 0.001%), 나머지 타입: 99.8–99.9%(연간 고장률 0.1–0.2%) |
| 루트 볼륨 | OS가 설치된 부팅 디스크. 인스턴스를 실행할 때 붙인 루트 볼륨은 기본적으로 **종료 시 함께 삭제** |
| 데이터 볼륨 | 나중에 붙인 볼륨은 기본적으로 인스턴스 종료 후에도 **남음**(그리고 계속 과금) |

**내구성(durability)**은 "데이터를 잃지 않을 확률"이다. 99.9%라도 1,000개 중 1개는 1년 안에 고장 날 수 있다는 뜻이다. 그래서 중요한 데이터는 **스냅샷으로 백업**해야 한다.

## 3단계: 다른 저장소와 무엇이 다른가

출처: [EC2 storage options](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Storage.html), [Instance store](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html)

| 저장소 | 종류 | 특징 | 언제 |
| --- | --- | --- | --- |
| **EBS** | 블록 | 인스턴스와 독립적으로 유지, AZ 단위, 스냅샷 백업 | OS 디스크, DB, 일반 애플리케이션 데이터 |
| **Instance store** | 블록 | 호스트 서버에 **물리적으로 붙은** 디스크. 매우 빠르지만 **인스턴스를 멈추거나 종료하면 데이터가 사라짐** | 캐시, 임시 파일, 복제되는 데이터 |
| **EFS** | 파일(NFS) | 여러 인스턴스가 **동시에** 마운트하는 공유 파일시스템(Linux) | 여러 서버가 같은 파일을 공유 |
| **S3** | 객체 | 용량 제한 없이 HTTP로 저장·조회 | 백업, 정적 파일, 데이터 레이크 |

EBS 볼륨은 기본적으로 **인스턴스 하나에만** 붙는다. 여러 서버가 같은 파일을 봐야 하면 EFS를 쓴다(io1·io2의 Multi-Attach라는 예외가 있지만 특수한 용도다 → [03](./03-operations.md#multi-attach)).

## 4단계: 볼륨 타입과 성능

디스크 성능은 보통 두 숫자로 본다.

| 지표 | 뜻 | 중요한 작업 |
| --- | --- | --- |
| **IOPS** | 초당 읽기·쓰기 횟수 | DB처럼 작은 데이터를 자주 읽고 쓰는 작업 |
| **처리량(throughput)** | 초당 읽고 쓰는 데이터 양(MiB/s) | 로그 처리, 대용량 파일 스캔처럼 큰 데이터를 연속으로 읽는 작업 |
| **지연(latency)** | I/O 하나가 끝날 때까지 걸리는 시간 | 응답 속도가 중요한 DB |

EBS는 용도에 따라 **볼륨 타입**을 고른다.

| 계열 | 타입 | 한 줄 요약 |
| --- | --- | --- |
| 범용 SSD | **gp3**, gp2 | 대부분의 작업. **gp3가 기본 선택**(gp2보다 GB당 20% 저렴) |
| Provisioned IOPS SSD | **io2**, io1 | 높은 IOPS와 낮은 지연이 필요한 핵심 DB |
| HDD | **st1**, sc1 | 큰 데이터를 연속으로 읽는 작업 / 거의 안 읽는 데이터. **부팅 디스크로 못 씀** |

인스턴스 자체에도 EBS와 주고받을 수 있는 최대 대역폭이 있다. 볼륨을 아무리 빠르게 설정해도 **인스턴스 타입의 EBS 한도**를 넘을 수는 없다. 자세한 내용은 [02-volume-types.md](./02-volume-types.md)에서 다룬다.

## 5단계: 스냅샷 — 백업과 복제

**스냅샷(snapshot)**은 볼륨의 특정 시점 사본이다.

- AWS가 관리하는 S3에 저장된다(내 S3 버킷에는 보이지 않음).
- **증분(incremental)** 방식이다. 첫 스냅샷은 전체를, 다음부터는 **바뀐 블록만** 저장한다.
- 스냅샷으로 **새 볼륨**을 만들 수 있다. 다른 AZ에도 만들 수 있고, 스냅샷을 다른 리전에 복사할 수도 있다. AZ에 묶인 볼륨을 옮기는 방법이 바로 이것이다.

```
볼륨(ap-northeast-2a) ──스냅샷──▶ 스냅샷(리전 전체) ──복원──▶ 새 볼륨(ap-northeast-2c)
                                         └──복사──▶ 다른 리전의 스냅샷
```

스냅샷에서 만든 볼륨은 **처음 읽는 블록이 느리다**(데이터를 S3에서 가져오기 때문). 자세한 내용은 [04-snapshots.md](./04-snapshots.md)에서 다룬다.

## 6단계: 암호화

EBS 볼륨은 **KMS**(AWS 암호화 키 관리 서비스) 키로 암호화할 수 있다.

- 암호화하면 저장 데이터, 인스턴스와 볼륨 사이의 전송 데이터, 스냅샷까지 모두 암호화된다.
- 성능 영향은 거의 없다.
- **리전 단위로 "기본 암호화"를 켜 두면** 이후 만드는 모든 볼륨이 자동으로 암호화된다.
- 이미 만든 암호화되지 않은 볼륨은 그 자리에서 암호화할 수 없다(스냅샷을 거쳐야 함).

자세한 내용은 [03-operations.md](./03-operations.md#암호화)에서 다룬다.

## 7단계: 요금은 어떻게 매겨지나

출처: [EBS pricing](https://aws.amazon.com/ebs/pricing/), AWS 가격 데이터(2026-10-12 게시). us-east-1 기준이며 괄호는 서울 값이다.

| 항목 | 단가 |
| --- | --- |
| gp3 용량 | GB-월 $0.08($0.0912). 3,000 IOPS·125 MiB/s 포함 |
| gp2 용량 | GB-월 $0.10($0.114) |
| io2 용량 + IOPS | GB-월 $0.125 + IOPS-월 $0.065부터 |
| st1 / sc1 | GB-월 $0.045 / $0.015 |
| 스냅샷 | GB-월 $0.05(서울 같음). 실제로 저장된(증분) 데이터 기준 |

핵심은 이렇다.

- **사용량이 아니라 만든 용량에 과금**된다. 100 GiB 볼륨에 10 GiB만 써도 100 GiB 요금을 낸다.
- **인스턴스가 꺼져 있어도, 볼륨이 어디에도 연결되지 않아도** 요금이 나간다.
- 스냅샷도 지우기 전까지 계속 과금된다.

비용 관리는 [05-cost-issues.md](./05-cost-issues.md)에서 다룬다.

## 용어 정리

| 용어 | 뜻 | 처음 나오는 곳 |
| --- | --- | --- |
| 블록 스토리지 | 데이터를 고정 크기 블록 단위로 읽고 쓰는 저장장치(디스크) | 1단계 |
| 파일시스템 | 디스크에 파일을 저장하는 방식(XFS, ext4 등) | 1단계 |
| EBS 볼륨 | 인스턴스에 붙이는 EBS 가상 디스크 | 1단계 |
| 가용 영역(AZ) | 리전 안의 독립된 데이터센터 묶음. EBS 볼륨의 범위 | 2단계 |
| 내구성 | 데이터를 잃지 않을 확률 | 2단계 |
| 루트 볼륨 / 데이터 볼륨 | OS 부팅 디스크 / 추가 데이터 디스크 | 2단계 |
| Instance store | 호스트에 물리적으로 붙은 임시 디스크 | 3단계 |
| EFS / S3 | 공유 파일시스템 / 객체 스토리지 | 3단계 |
| IOPS / 처리량 / 지연 | 초당 I/O 횟수 / 초당 데이터 양 / I/O 하나의 소요 시간 | 4단계 |
| 볼륨 타입 | gp3·gp2·io2·io1·st1·sc1 등 성능·가격 등급 | 4단계 |
| 스냅샷 | 볼륨의 특정 시점 사본. S3에 증분 저장 | 5단계 |
| 증분 | 바뀐 부분만 저장하는 방식 | 5단계 |
| KMS | AWS 암호화 키 관리 서비스 | 6단계 |
| GB-월 | 1 GB를 한 달 동안 유지하는 요금 단위 | 7단계 |

## 출처

- https://docs.aws.amazon.com/ebs/latest/userguide/what-is-ebs.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Storage.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/preserving-volumes-on-termination.html
- https://aws.amazon.com/ebs/pricing/

---
*조사 기준일: 2026-10-14. 요금은 AWS 요금 페이지가 쓰는 가격 데이터(2026-10-12 게시) 기준이다. 다이어그램은 이해를 돕기 위한 예시다.*
