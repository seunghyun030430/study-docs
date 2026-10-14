# 스냅샷과 백업

> 스냅샷은 **증분**으로 저장되므로 스냅샷 하나를 지운다고 그만큼 비용이 줄지 않고, 복원에는 **가장 최근 스냅샷 하나만** 있으면 된다. 스냅샷은 실행 중에도 찍을 수 있지만 **OS·앱 캐시는 담기지 않는다.** 복원한 볼륨은 **처음 읽는 블록이 느리므로** 운영 전에 초기화한다. 자동화는 EBS만 다루고 단일 계정이면 **Data Lifecycle Manager(무료)**, 여러 서비스·여러 계정을 중앙에서 관리하면 **AWS Backup**을 쓴다. 실수로 지우는 것은 **Recycle Bin**과 **snapshot lock**으로 막는다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md) 5단계(스냅샷, 증분)와 [03-operations.md](./03-operations.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Crash-consistent | 전원이 갑자기 꺼진 순간과 같은 상태. 디스크에 쓰인 데이터만 담기고 메모리의 내용은 빠짐 |
| Application-consistent | 앱이 쓰기를 멈추고 메모리 내용을 디스크에 내려쓴 뒤 찍은 상태. DB를 그대로 복구할 수 있음 |
| VSS | Windows의 볼륨 섀도 복사 서비스. 앱 일관성 스냅샷에 씀 |
| Lazy loading | 스냅샷으로 만든 볼륨의 블록을 처음 읽을 때 S3에서 가져오는 방식 |
| Fast Snapshot Restore(FSR) | 미리 초기화된 상태로 볼륨을 만들어 주는 유료 기능 |
| DSU(Data Services Unit) | FSR 과금 단위. 스냅샷 1개 × AZ 1개 = 1 DSU |
| Archive tier | 거의 안 쓰는 스냅샷을 싸게 장기 보관하는 저장 계층 |
| Recycle Bin | 삭제한 스냅샷·AMI·볼륨을 일정 기간 보관해 복구할 수 있게 하는 기능 |
| AMI | EC2 인스턴스를 만드는 이미지. EBS 기반 AMI는 내부적으로 스냅샷을 참조 |
| Data Lifecycle Manager(DLM) | 스냅샷·AMI의 생성·보관·삭제를 정책으로 자동화하는 EBS 기능 |

## 스냅샷이 저장되는 방식

출처: [EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html), [How snapshots work](https://docs.aws.amazon.com/ebs/latest/userguide/how_snapshots_work.html)

```
볼륨 200 GiB (실제 데이터 50 GiB)

스냅샷 A  [■■■■■ 50 GiB]               ← 첫 스냅샷: 쓰인 데이터 전체
스냅샷 B  [  △ 5 GiB 변경분 ]           ← 바뀐 블록만
스냅샷 C  [   ▲ 3 GiB 변경분 ]          ← 바뀐 블록만

스냅샷 B를 지우면?
→ B에만 있던 블록 중 C가 참조하지 않는 것만 삭제됨
→ C로 복원하면 여전히 전체 데이터가 나옴
```

| 성질 | 내용 |
| --- | --- |
| 저장 위치 | AWS가 관리하는 S3. **내 S3 콘솔·API에서는 보이지 않음** |
| 범위 | 리전 안의 모든 AZ에 복제. 스냅샷으로 같은 리전의 어느 AZ에든 볼륨 생성 가능 |
| 크기 | 볼륨 크기가 아니라 **쓰인 데이터** 기준. 200 GiB 볼륨에 50 GiB가 있으면 첫 스냅샷은 50 GiB |
| 증분 | 이후 스냅샷은 바뀐 블록만 저장 |
| 복원 | **가장 최근 스냅샷 하나**만 있으면 전체 볼륨 복원 가능 |
| 삭제 효과 | 그 스냅샷만 참조하던 데이터만 지워짐. **삭제해도 비용이 크게 줄지 않을 수 있음** |
| 전체 스냅샷이 되는 경우 | 다른 계정에서 공유받은 스냅샷 기반이거나, 볼륨이 다른 KMS 키를 쓰면 다음 스냅샷은 전체 |

**AWS는 EBS 볼륨을 자동으로 백업해 주지 않는다.** 스냅샷은 직접 만들거나 DLM·AWS Backup으로 자동화해야 한다.

## 스냅샷 만들기

출처: [Create snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-snapshot.html), [Multi-volume snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-create-snapshots.html)

```bash
# 볼륨 하나
aws ec2 create-snapshot --volume-id vol-0123456789abcdef0 \
  --description "app-data before upgrade" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=app-data-20261014}]'

# 인스턴스에 붙은 볼륨 여러 개를 같은 시점으로 (최대 128개)
aws ec2 create-snapshots \
  --instance-specification InstanceId=i-0123456789abcdef0,ExcludeBootVolume=false \
  --copy-tags-from-source volume
```

### 일관성: 무엇이 담기나

| 방식 | 담기는 것 | 방법 |
| --- | --- | --- |
| 그냥 찍기 | 요청 시점에 디스크에 쓰인 데이터만(**캐시 제외**) | `create-snapshot` |
| 여러 볼륨 동시 | 인스턴스 전체가 **crash-consistent** | `create-snapshots`(RAID·LVM으로 묶은 볼륨에 필수) |
| 쓰기 멈추고 찍기 | 앱 데이터가 온전히 내려써진 상태 | 쓰기 중지·`fsfreeze`·언마운트 후 찍기, 루트 볼륨은 인스턴스 정지 권장 |
| Windows 앱 일관성 | VSS로 앱을 잠시 멈추고 찍음 | SSM 문서 `AWSEC2-CreateVssSnapshot` |
| DLM 사전·사후 스크립트 | Linux 앱을 멈췄다 재개 | DLM custom policy의 pre/post script |

스냅샷은 완료될 때까지 `pending` 상태이고, 그동안에도 볼륨은 계속 쓸 수 있다. **DB 볼륨은 DB의 백업 기능을 쓰거나, 쓰기를 멈추고 찍는다.** 그냥 찍은 스냅샷으로 복원하면 DB가 복구(recovery) 과정을 거쳐야 할 수 있다.

## 복원과 초기화

출처: [Initialize volumes](https://docs.aws.amazon.com/ebs/latest/userguide/initalize-volume.html), [Fast Snapshot Restore](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html)

```bash
aws ec2 create-volume --snapshot-id snap-0123456789abcdef0 \
  --availability-zone ap-northeast-2c --volume-type gp3
```

스냅샷으로 만든 볼륨은 바로 쓸 수 있지만, **블록을 처음 읽을 때 S3에서 가져오므로(lazy loading) 그 블록은 느리다.** 복원 직후 DB를 띄우면 한동안 매우 느릴 수 있다.

| 방법 | 동작 | 비용 | 언제 |
| --- | --- | --- | --- |
| 그대로 사용 | 처음 읽을 때마다 가져옴 | 없음 | 급하지 않은 볼륨 |
| `fio`/`dd`로 전체 읽기 | 직접 모든 블록을 한 번 읽음 | 없음(시간이 걸림) | 운영 투입 전 |
| **Provisioned Rate for Volume Initialization** | 100–300 MiB/s 중 원하는 속도로 초기화 보장 | 스냅샷 전체 데이터 GB당 과금(예시 $0.0036/GB) | 초기화 완료 시간을 예측해야 할 때 |
| **Fast Snapshot Restore** | 만들자마자 완전히 초기화된 볼륨 | DSU-시간당 $0.75(서울 $0.855) | 같은 스냅샷으로 자주·빨리 볼륨을 만들 때(오토스케일링 등) |

```bash
# 초기화 속도 지정 (스냅샷에서 만들 때만)
aws ec2 create-volume --snapshot-id snap-0123456789abcdef0 \
  --availability-zone ap-northeast-2c --volume-type gp3 \
  --volume-initialization-rate 300

# 수동 초기화 (읽기만 하므로 데이터는 바뀌지 않음)
sudo fio --filename=/dev/nvme1n1 --rw=read --bs=1M --iodepth=32 \
  --ioengine=libaio --direct=1 --name=volume-initialize
```

- Provisioned Rate는 **볼륨 크기가 아니라 스냅샷의 전체 데이터 크기** 기준으로 시간과 비용이 정해진다. 볼륨이 활성화되면 전액 과금되므로, 초기화가 끝나기 전에 지워도 환불되지 않는다. 리전당 동시 요청 합계는 5,000 MiB/s이고, FSR보다 우선 적용된다.
- FSR은 **스냅샷 × AZ 쌍마다** 켜고, 분 단위(최소 1시간)로 과금된다. 예를 들어 스냅샷 1개를 AZ 1곳에서 한 달(720시간) 켜 두면 720 × $0.75 = **$540**이다.
- FSR에는 크레딧 개념이 있어 볼륨을 한꺼번에 많이 만들면 일부는 FSR 효과를 받지 못한다. 기본 한도는 리전당 FSR 스냅샷 5개, 스냅샷 크기 16 TiB까지다.

## 복사와 공유

출처: [Copy a snapshot](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-copy-snapshot.html), [Share a snapshot](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-modifying-snapshot-permissions.html), [Block public access](https://docs.aws.amazon.com/ebs/latest/userguide/block-public-access-snapshots.html)

```bash
# 다른 리전으로 복사 (재해 복구용)
aws ec2 copy-snapshot --region ap-northeast-1 \
  --source-region ap-northeast-2 --source-snapshot-id snap-0123456789abcdef0 \
  --encrypted --kms-key-id alias/dr-ebs

# 다른 계정과 공유
aws ec2 modify-snapshot-attribute --snapshot-id snap-0123456789abcdef0 \
  --attribute createVolumePermission --operation-type add --user-ids 444455556666
```

| 항목 | 내용 |
| --- | --- |
| 증분 복사 | 같은 계정·리전·KMS 키면 항상 증분. 다른 리전·계정은 대상에 이전 복사본이 있고 같은 키일 때만 증분, 아니면 **전체 복사** |
| 태그 | 사용자 태그는 복사되지 않음 |
| 시간 지정 복사 | 완료 시간을 15분–48시간(15분 단위)으로 지정 가능. 복사당 500 MiB/s |
| 동시 복사 | 대상 리전당 20개 |
| 비용 | 리전 간 복사는 데이터 전송 요금 발생 |
| **공유 제약** | **`aws/ebs` 키로 암호화한 스냅샷은 공유할 수 없음.** 고객 관리형 키를 쓰고 그 키도 공유해야 함 |
| 공개 공유 | 암호화되지 않은 스냅샷만 가능. 계정 설정 **Block public access**(리전 단위)로 막을 수 있음 |

공개 스냅샷은 데이터 유출 사고의 흔한 원인이다. 모든 리전에서 **Block public access를 켜 두는 것**이 안전하다. 이 설정은 AMI 공개는 막지 않으므로 AMI용 block public access도 따로 켠다.

## 장기 보관: Archive tier

출처: [Archive snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive.html), [Archive considerations](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive-considerations.html)

| 항목 | Standard | Archive |
| --- | --- | --- |
| 저장 요금 | GB-월 $0.05 | GB-월 $0.0125 |
| 형태 | 증분 | **전체 스냅샷으로 변환** |
| 최소 보관 | 없음 | **90일**(일찍 지우면 남은 일수만큼 과금) |
| 복원 | 즉시 | **최대 72시간**, GB당 $0.03 |
| 할 수 없는 것 | — | 볼륨 생성, 복사, 공유, FSR(복원 후 가능) |

```bash
aws ec2 modify-snapshot-tier --snapshot-id snap-0123456789abcdef0 --storage-tier archive
aws ec2 restore-snapshot-tier --snapshot-id snap-0123456789abcdef0 --temporary-restore-days 7
```

- AWS는 90일 이상 보관하는 스냅샷에서 **최대 75% 저렴**하다고 안내한다.
- **매일 찍는 증분 스냅샷을 archive로 옮기면 오히려 비쌀 수 있다.** 각각이 전체 스냅샷으로 바뀌기 때문이다. 월·분기·연 단위의 규정 보관용 스냅샷에 쓴다.
- 공유 중인 스냅샷을 archive하면 다른 계정의 접근이 끊긴다. 시작한 archive·복원은 취소할 수 없다.

## 실수로 지우는 것 막기

| 기능 | 하는 일 | 비용 | 출처 |
| --- | --- | --- | --- |
| **Recycle Bin** | 지운 스냅샷·AMI·볼륨을 보관 기간 동안 보관해 복구 가능(스냅샷·AMI 1–365일, 볼륨 1–7일). 태그 단위나 리전 단위 규칙 | 기능 무료, 보관 중인 리소스는 원래 요금 | [Recycle Bin](https://docs.aws.amazon.com/ebs/latest/userguide/recycle-bin.html) |
| Recycle Bin rule lock | 규칙 자체를 잠가 해제 대기(7–30일) 없이는 못 바꾸게 함(리전 단위 규칙만) | 무료 | [Rule lock](https://docs.aws.amazon.com/ebs/latest/userguide/recycle-bin-lock.html) |
| **Snapshot lock** | 스냅샷을 WORM 방식으로 잠가 **아무도 삭제 못 하게**(governance·compliance 모드) | 무료 | [Snapshot lock](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshot-lock.html) |

```bash
aws rbin create-rule --resource-type EBS_SNAPSHOT \
  --retention-period RetentionPeriodValue=14,RetentionPeriodUnit=DAYS \
  --description "keep deleted snapshots 14 days"
```

`create-rule` 명령은 Recycle Bin 문서의 개념으로 조합한 예시이며, CLI 레퍼런스와 직접 대조하지 않았다(확인 필요). 랜섬웨어처럼 권한을 탈취한 공격자가 백업까지 지우는 상황에 대비하려면, snapshot lock의 compliance 모드나 AWS Backup Vault Lock을 쓴다.

## 자동화: DLM vs AWS Backup

### Data Lifecycle Manager

출처: [Data Lifecycle Manager](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html), [Default vs custom policies](https://docs.aws.amazon.com/ebs/latest/userguide/policy-differences.html)

**추가 요금이 없다**(스냅샷 저장 요금만 냄). DLM이 만들지 않은 스냅샷·AMI는 관리하지 않는다.

| 항목 | 기본 정책(default) | 사용자 정책(custom) |
| --- | --- | --- |
| 대상 | 최근 스냅샷이 없는 모든 볼륨 | **태그**로 고른 볼륨·인스턴스 |
| 일정 | 1개, 1–7일마다 | 정책당 최대 4개, 매일·매주·매월·매년·cron |
| 보관 | 기간 2–14일 | 개수 최대 1,000개 또는 기간 최대 100년 |
| 앱 일관성 스크립트 | 불가 | 가능 |
| Archive·FSR·계정 간 공유 | 불가 | 가능 |
| 리전 간 복사 | 기본 설정 | 최대 3개 리전, 대상별 보관 기간 |

리전당 사용자 정책 100개, 기본 정책은 스냅샷용·AMI용 각 1개까지 만들 수 있다.

```bash
# 태그 Backup=daily 볼륨을 매일 03:00(UTC)에 찍고 7개 보관하는 정책
aws dlm create-lifecycle-policy \
  --description "daily-7" --state ENABLED \
  --execution-role-arn arn:aws:iam::111122223333:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [{"Key": "Backup", "Value": "daily"}],
    "Schedules": [{
      "Name": "daily",
      "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS", "Times": ["03:00"]},
      "RetainRule": {"Count": 7},
      "CopyTags": true
    }]
  }'
```

이 JSON은 DLM 정책 구조를 따라 이 레포에서 작성한 예시이며, 실제 계정에서 실행하지 않았다(확인 필요). `AWSDataLifecycleManagerDefaultRole`은 콘솔에서 처음 정책을 만들 때 생기는 기본 role이다.

### AWS Backup

출처: [What is AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html), [Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)

- EBS뿐 아니라 RDS, DynamoDB, EFS 등 여러 서비스를 **하나의 백업 계획**으로 관리한다.
- 태그로 대상을 고르고, 리전 간 복사와 **Organizations 기반 계정 간 백업**을 지원한다.
- **Vault Lock**(compliance 모드)을 걸면 유예 기간이 지난 뒤에는 AWS를 포함해 누구도 백업을 지우거나 정책을 바꿀 수 없다.
- EBS는 기본적으로 다중 볼륨 crash-consistent 스냅샷으로 백업한다.
- AWS Backup이 만든 스냅샷은 AWS Backup을 통해서만 지울 수 있다.

### 무엇을 쓸까

| 상황 | 추천 |
| --- | --- |
| EBS·AMI만, 한 계정, 비용 0으로 시작 | **DLM** |
| 여러 서비스(RDS, EFS 등)를 함께, 조직 전체 정책, 감사·규정 준수 | **AWS Backup** |
| 랜섬웨어 대비 변경 불가 백업 | AWS Backup Vault Lock(compliance) 또는 snapshot lock |

AWS가 둘 중 무엇을 쓰라고 직접 권장하는 문서는 찾지 못했다. 위 표는 두 서비스의 기능 차이로 정리한 것이다.

## 한도

출처: [EBS quotas](https://docs.aws.amazon.com/general/latest/gr/ebs-service.html)

| 항목 | 기본값 | 증설 |
| --- | --- | --- |
| 리전당 스냅샷 | 100,000 | 가능 |
| 볼륨당 동시 스냅샷 | 5개(SSD·standard), 1개(st1·sc1) | 불가 |
| 대상 리전당 동시 복사 | 20 | Support 요청 |
| FSR 스냅샷 | 5 | 가능 |
| 볼륨당 archive 스냅샷 / 진행 중 archive / 진행 중 복원 | 25 / 25 / 5 | 가능 |

## 이슈 / 주의

- **AMI를 등록 해제해도 스냅샷은 남는다.** `aws ec2 deregister-image --image-id ami-... --delete-associated-snapshots`로 함께 지운다(다른 AMI가 쓰는 스냅샷은 남김). 반대로 AMI가 등록된 동안에는 그 루트 스냅샷을 지울 수 없다. ([Deregister an AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/deregister-ami.html))
- **스냅샷 하나를 지워도 비용이 별로 안 줄어든다.** 증분이라 다른 스냅샷이 참조하는 블록은 남기 때문이다. 비용을 줄이려면 보관 개수·기간 정책을 줄인다.
- **복원 직후 성능 저하**를 장애로 오해하지 않는다. 초기화를 먼저 한다.
- **`aws/ebs` 키로 암호화하면 나중에 계정 간 공유·DR이 막힌다.** 처음부터 고객 관리형 키를 쓰면 이 문제가 없다.
- **daily 증분 스냅샷의 archive 이동**은 비용을 늘릴 수 있다.

## 출처

- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html
- https://docs.aws.amazon.com/ebs/latest/userguide/how_snapshots_work.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-snapshot.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-create-snapshots.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/application-consistent-snapshots.html
- https://docs.aws.amazon.com/ebs/latest/userguide/initalize-volume.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-copy-snapshot.html
- https://docs.aws.amazon.com/ebs/latest/userguide/time-based-copies.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-modifying-snapshot-permissions.html
- https://docs.aws.amazon.com/ebs/latest/userguide/block-public-access-snapshots.html
- https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive.html
- https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive-considerations.html
- https://docs.aws.amazon.com/ebs/latest/userguide/recycle-bin.html
- https://docs.aws.amazon.com/ebs/latest/userguide/recycle-bin-lock.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshot-lock.html
- https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html
- https://docs.aws.amazon.com/ebs/latest/userguide/policy-differences.html
- https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html
- https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html
- https://docs.aws.amazon.com/general/latest/gr/ebs-service.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/deregister-ami.html
- https://aws.amazon.com/ebs/pricing/

---
*조사 기준일: 2026-10-14. 요금은 AWS 요금 페이지가 쓰는 가격 데이터(2026-10-12 게시) 기준이다. CLI 예시는 실제 계정에서 실행하지 않았다.*
