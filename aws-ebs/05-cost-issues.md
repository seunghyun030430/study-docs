# 비용·한도·운영 이슈

> EBS 비용은 **"만들어 둔 것"**에 매겨진다. 그래서 낭비의 대부분은 **연결되지 않은 볼륨**, **AMI를 지운 뒤 남은 스냅샷**, **과하게 잡은 용량·IOPS**, **아직 gp2인 볼륨**에서 나온다. 먼저 gp2를 gp3로 바꾸고(GB당 20% 저렴, 무중단), 버려진 볼륨·스냅샷을 정기적으로 찾아 지우고, Compute Optimizer로 크기·성능을 맞춘다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[04](./04-snapshots.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Orphaned(버려진) 리소스 | 어디에도 쓰이지 않는데 남아서 과금되는 볼륨·스냅샷 |
| Rightsizing | 실제 사용량에 맞춰 용량·성능을 줄이거나 늘리는 것 |
| Trusted Advisor | AWS 계정의 비용·보안·성능 점검 결과를 보여 주는 서비스 |
| Compute Optimizer | 사용 메트릭을 분석해 EC2·EBS 등의 적정 설정을 추천하는 서비스 |
| Cost Optimization Hub | 여러 계정·리전의 비용 절감 추천을 한곳에 모아 보여 주는 기능 |

## 요금 구조

출처: [EBS pricing](https://aws.amazon.com/ebs/pricing/), AWS 요금 페이지 가격 데이터(2026-10-12 게시)

### 볼륨

| 타입 | 용량(GB-월) us-east-1 | 서울 | 성능 요금 |
| --- | --- | --- | --- |
| gp3 | $0.08 | $0.0912 | 3,000 IOPS·125 MiB/s 포함. 초과 IOPS-월 $0.005(서울 $0.0057), 초과 처리량 MiB/s-월 약 $0.04(서울 약 $0.0456) |
| gp2 | $0.10 | $0.114 | 용량에 포함(크기에 비례) |
| io2 | $0.125 | $0.1278 | IOPS-월: 첫 32,000개 $0.065 → 64,000개까지 $0.0455 → 그 이상 $0.03185 |
| io1 | $0.125 | $0.1278 | IOPS-월 $0.065 |
| st1 | $0.045 | $0.051 | 포함 |
| sc1 | $0.015 | $0.0174 | 포함 |

- gp3 처리량 단가는 가격 데이터 기준 us-east-1 값이다. 요금 페이지 본문 예시는 리전을 밝히지 않고 MB/s-월 $0.06을 써서 값이 다르다(확인 필요).
- io2 두 번째 구간 단가는 요금 페이지 HTML에서 $0.046, 가격 데이터에서 $0.0455로 조금 다르다.

### 스냅샷과 부가 기능

| 항목 | us-east-1 | 서울 |
| --- | --- | --- |
| 스냅샷(Standard) | GB-월 $0.05 | $0.05 |
| 스냅샷(Archive) | GB-월 $0.0125 | $0.0125 |
| Archive 복원 | GB당 $0.03 | $0.03 |
| Fast Snapshot Restore | DSU-시간 $0.75 | $0.855 |
| 볼륨 초기화 속도 지정 | 스냅샷 데이터 GB당 $0.0036(요금 페이지 예시) | 확인 필요 |
| DLM, Recycle Bin, snapshot lock, Block public access | 무료(보관 중인 스냅샷은 원래 요금) | 같음 |

## 계산해 보기

us-east-1 단가로 계산한 예시다.

**예 1: 500 GiB 볼륨, 6,000 IOPS가 필요한 DB**

| 선택 | 계산 | 월 비용 |
| --- | --- | --- |
| gp2(크기로 IOPS 확보) | 500 GiB면 1,500 IOPS뿐 → 2,000 GiB로 키워야 6,000 IOPS. 2,000 × $0.10 | **$200** |
| gp3 | 500 × $0.08 + (6,000 − 3,000) × $0.005 | 40 + 15 = **$55** |
| io2 | 500 × $0.125 + 6,000 × $0.065 | 62.5 + 390 = **$452.5** |

io2는 낮은 지연과 99.999% 내구성이 꼭 필요할 때만 쓴다.

**예 2: 매일 스냅샷, 30개 보관(볼륨 데이터 200 GiB, 하루 변경 2%)**

```
첫 스냅샷 200 GiB + 29일 × 4 GiB 변경분 = 약 316 GiB
316 × $0.05 ≈ 월 $15.8
```

같은 데이터를 매일 **전체** 복사했다면 30 × 200 = 6,000 GiB, 월 $300이다. 증분 저장 덕분에 훨씬 싸다. 반대로 변경이 많은 볼륨(로그, 임시 파일)은 스냅샷 비용도 빠르게 는다.

## gp2 → gp3 전환

출처: [General Purpose SSD](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html), [Modify a volume](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-modify-volume.html)

> "gp3 volumes offer a 20 percent lower price per GiB than General Purpose SSD (gp2) volumes."

```bash
# gp2 볼륨 목록
aws ec2 describe-volumes --filters Name=volume-type,Values=gp2 \
  --query 'Volumes[].[VolumeId,Size,Iops,State]' --output table

# 무중단 전환 (IOPS·처리량을 생략하면 gp2 성능과 gp3 기본값 중 높은 쪽 유지)
aws ec2 modify-volume --volume-id vol-0123456789abcdef0 --volume-type gp3
```

| 확인할 것 | 이유 |
| --- | --- |
| 1 TiB 이상 gp2 | 기본 IOPS가 3,000을 넘는다(GiB당 3). gp3로 바꿀 때 그만큼 IOPS를 지정해야 함(자동으로 높은 쪽 유지) |
| 처리량 | gp2 큰 볼륨은 최대 250 MiB/s. gp3 기본은 125이므로 필요하면 올림 |
| 변경 횟수 | 24시간에 4번까지 |
| 콘솔·IaC 기본값 | CLI·API 기본 타입은 여전히 **gp2**. 템플릿·스크립트의 기본값도 gp3로 고친다 |

## 버려진 리소스 찾기

### 연결되지 않은 볼륨

```bash
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].[VolumeId,Size,VolumeType,CreateTime]' --output table
```

- `available` 상태는 **어디에도 연결되지 않았지만 계속 과금되는** 볼륨이다. ([Delete a volume](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-deleting-volume.html))
- 주된 원인은 `DeleteOnTermination=false`이거나, 실행 후에 붙인 볼륨이 인스턴스 종료 뒤 남은 경우다([03](./03-operations.md#인스턴스-종료-시-볼륨-삭제deleteontermination)).
- 지우기 전에 필요하면 스냅샷을 남긴다. **볼륨 삭제는 되돌릴 수 없다**(Recycle Bin 규칙이 있으면 1–7일 복구 가능).

```bash
aws ec2 create-snapshot --volume-id vol-0123456789abcdef0 --description "before delete"
aws ec2 delete-volume --volume-id vol-0123456789abcdef0
```

### 남은 스냅샷

```bash
# 내 계정 스냅샷 중 오래된 것 (2026-01-01 이전)
aws ec2 describe-snapshots --owner-ids self \
  --query 'Snapshots[?StartTime<`2026-01-01`].[SnapshotId,VolumeSize,StartTime,Description]' \
  --output table
```

- AMI를 등록 해제할 때 `--delete-associated-snapshots`를 쓰지 않으면 스냅샷이 남는다([04](./04-snapshots.md#이슈--주의)).
- 원본 볼륨이 이미 없는 스냅샷(`VolumeId`가 존재하지 않는 볼륨)도 정리 후보다.
- 자동화 도구(DLM, AWS Backup)가 만든 스냅샷은 그 도구의 보관 정책으로 관리한다.

### AWS 도구

| 도구 | 하는 일 | 출처 |
| --- | --- | --- |
| Trusted Advisor "Underutilized Amazon EBS volumes" | 연결되지 않았거나 7일 동안 하루 1 IOPS 미만인 볼륨 | [Cost optimization checks](https://docs.aws.amazon.com/awssupport/latest/user/cost-optimization-checks.html) |
| Trusted Advisor "Amazon EBS over-provisioned volumes" | Compute Optimizer 데이터로 과하게 잡은 볼륨 | 같음 |
| Compute Optimizer | 14일 메트릭으로 볼륨 타입·크기·IOPS·처리량 추천(매일 갱신) | [EBS recommendations](https://docs.aws.amazon.com/compute-optimizer/latest/ug/view-ebs-recommendations.html) |
| Cost Optimization Hub | 여러 계정·리전의 rightsizing·유휴 리소스 추천을 모아 봄 | [Cost Optimization Hub](https://docs.aws.amazon.com/cost-management/latest/userguide/cost-optimization-hub.html) |

## 주요 한도

출처: [EBS quotas](https://docs.aws.amazon.com/general/latest/gr/ebs-service.html)

| 항목(리전당 기본값) | 값 | 증설 |
| --- | --- | --- |
| gp3 총 용량 | 50 TiB(서울·us-east-1 포함 대부분 리전) | 가능 |
| gp2·io1·st1·sc1 총 용량 | 50 TiB(서울 포함 주요 16개 리전), 그 외 300 TiB | 가능 |
| io2 총 용량 | 20 TiB | 가능 |
| io1 / io2 총 IOPS | 300,000 / 100,000 | 가능 |
| 스냅샷 수 | 100,000 | 가능 |
| 볼륨 초기화 속도 합계 | 5,000 MiB/s | 가능 |

**총 용량 한도가 생각보다 작다**(gp3 50 TiB). 대규모 마이그레이션 전에 Service Quotas에서 미리 늘린다. EBS가 사용량을 보고 자동으로 한도를 올려 주기도 한다.

## 흔한 장애와 함정

| 증상 | 원인 | 대응 | 자세히 |
| --- | --- | --- | --- |
| 볼륨을 다른 인스턴스에 붙일 수 없음 | 다른 AZ | 스냅샷 → 대상 AZ에 볼륨 생성 | [01](./01-overview.md#5단계-스냅샷--백업과-복제) |
| 평소엔 빠른데 배치 중 갑자기 느려짐 | gp2·st1·sc1 burst 크레딧 소진, 작은 인스턴스의 30분 burst 소진 | `BurstBalance`·`EBSIOBalance%` 확인, gp3·큰 인스턴스로 | [02](./02-volume-types.md) |
| IOPS를 올렸는데 그대로 | 인스턴스 EBS 한도, 큰 I/O로 처리량 한도 | `InstanceEBS*ExceededCheck`, I/O 크기 확인 | [02](./02-volume-types.md#인스턴스-한도-볼륨보다-먼저-막힐-수-있다) |
| 복원한 볼륨에서 DB가 매우 느림 | lazy loading | fio 초기화, FSR, 초기화 속도 지정 | [04](./04-snapshots.md#복원과-초기화) |
| 볼륨을 늘렸는데 `df`에 그대로 | 파티션·파일시스템 미확장 | `growpart` + `xfs_growfs`/`resize2fs` | [03](./03-operations.md#늘린-크기를-os에-반영) |
| 볼륨 변경이 거부됨 | 24시간 4회 제한, 이전 변경 미완료, 크기 축소 시도 | 완료 후 재시도, 축소는 새 볼륨으로 복사 | [03](./03-operations.md#elastic-volumes-크기타입성능-바꾸기) |
| 재부팅 후 부팅 실패 | `/etc/fstab` 오류, 장치 이름 변경 | UUID + `nofail`, `mount -a`로 검증 | [03](./03-operations.md#포맷하고-마운트) |
| 스냅샷을 다른 계정에 공유 못 함 | `aws/ebs` 키로 암호화 | 고객 관리형 키로 복사 후 공유 | [04](./04-snapshots.md#복사와-공유) |
| 볼륨 상태 `impaired` | 데이터 불일치 가능성으로 I/O 차단 | `enable-volume-io` 후 `fsck` | [03](./03-operations.md#볼륨-상태-검사) |
| 볼륨이 `busy`에서 안 떨어짐 | 마운트된 채 분리 | 언마운트 후 재시도, force detach는 최후 수단 | [03](./03-operations.md#분리detach) |

## 안티패턴

| 안티패턴 | 문제 | 대안 |
| --- | --- | --- |
| 새 볼륨을 gp2로 생성(CLI 기본값) | 같은 성능에 20% 더 비쌈 | 스크립트·IaC에 `gp3` 명시 |
| IOPS를 얻으려고 gp2 크기를 키움 | 쓰지 않는 용량에 과금 | gp3에서 IOPS만 추가 |
| 필요 이상으로 io2 사용 | IOPS 요금이 큼 | gp3로 충분한지 먼저 확인 |
| 스냅샷 없이 운영 | AZ 장애·실수 삭제 시 복구 불가 | DLM·AWS Backup으로 자동 스냅샷 |
| 스냅샷 보관 정책 없음 | 스냅샷이 무한히 쌓임 | 개수·기간 보관 규칙 |
| 공개 스냅샷 허용 | 데이터 유출 | 모든 리전에 Block public access |
| 모든 볼륨에 `DeleteOnTermination=false` | 종료 후 볼륨이 쌓임 | 꼭 남길 볼륨만 false, 나머지는 정기 점검 |
| 용량을 넉넉히 미리 잡음 | 쓰지 않는 GB에 과금. 줄일 수도 없음 | 작게 시작해 Elastic Volumes로 늘림 |

## 출처

- https://aws.amazon.com/ebs/pricing/
- https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-modify-volume.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-deleting-volume.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/deregister-ami.html
- https://docs.aws.amazon.com/awssupport/latest/user/cost-optimization-checks.html
- https://docs.aws.amazon.com/compute-optimizer/latest/ug/view-ebs-recommendations.html
- https://docs.aws.amazon.com/cost-management/latest/userguide/cost-optimization-hub.html
- https://docs.aws.amazon.com/general/latest/gr/ebs-service.html

---
*조사 기준일: 2026-10-14. 요금은 AWS 요금 페이지가 쓰는 가격 데이터(2026-10-12 게시) 기준이고, 계산 예시는 그 단가로 이 레포에서 계산했다. `describe-volumes`의 `status` 필터 이름은 CLI 레퍼런스와 직접 대조하지 않았다(확인 필요).*
