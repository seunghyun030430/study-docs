# 운영: 연결·변경·암호화·Multi-Attach·모니터링

> 볼륨은 **같은 AZ의 인스턴스에만** 연결되고, Nitro 인스턴스에서는 `/dev/nvme1n1` 같은 NVMe 이름으로 보이므로 **볼륨 ID로 장치를 확인**한 뒤 마운트한다. 크기·타입·성능은 **Elastic Volumes**로 서비스를 멈추지 않고 바꾸되, **줄일 수는 없고 24시간에 4번**까지만 바꿀 수 있다. 크기를 늘린 뒤에는 **파티션과 파일시스템도 직접 늘려야** 한다. 암호화는 리전마다 **기본 암호화**를 켜 두는 것이 가장 간단하다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)와 [02-volume-types.md](./02-volume-types.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| NVMe | 빠른 저장장치용 인터페이스 규격. Nitro 인스턴스는 EBS 볼륨을 NVMe 장치로 보여 줌 |
| 장치 이름(device name) | OS가 디스크를 부르는 이름(`/dev/sdf`, `/dev/nvme1n1`) |
| 파일시스템 | 디스크에 파일을 저장하는 방식(XFS, ext4 등). 볼륨을 만들고 처음 한 번 생성(포맷) |
| 마운트 | 파일시스템을 특정 디렉터리(`/data`)에 연결하는 것 |
| `/etc/fstab` | 부팅할 때 자동으로 마운트할 목록을 적는 Linux 설정 파일 |
| UUID | 파일시스템마다 붙는 고유 ID. 장치 이름이 바뀌어도 그대로 |
| 파티션 | 디스크를 나눈 구역. 루트 볼륨은 보통 파티션이 있음(`nvme0n1p1`) |
| Elastic Volumes | 연결된 상태에서 볼륨 크기·타입·성능을 바꾸는 기능 |
| 클러스터 파일시스템 | 여러 서버가 동시에 같은 디스크를 써도 깨지지 않게 설계된 파일시스템(GFS2 등) |

## 만들고 연결하기

출처: [Create a volume](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html), [create-volume](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-volume.html), [attach-volume](https://docs.aws.amazon.com/cli/latest/reference/ec2/attach-volume.html)

```bash
# 1. 인스턴스와 같은 AZ에 gp3 볼륨 생성
VOL=$(aws ec2 create-volume \
  --availability-zone ap-northeast-2a \
  --volume-type gp3 --size 100 \
  --iops 3000 --throughput 125 \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=app-data}]' \
  --query VolumeId --output text)

# 2. available 상태가 될 때까지 기다린 뒤 연결
aws ec2 wait volume-available --volume-ids "$VOL"
aws ec2 attach-volume --volume-id "$VOL" --instance-id i-0123456789abcdef0 --device /dev/sdf
```

- **CLI·API·SDK의 기본 볼륨 타입은 `gp2`**다(콘솔은 `gp3`). CLI에서는 `--volume-type gp3`을 꼭 적는다.
- `--kms-key-id`를 쓰려면 `--encrypted`도 함께 넣어야 한다. 키를 생략하면 계정의 기본 EBS 키를 쓴다.
- 볼륨과 인스턴스는 **같은 AZ**여야 한다. 볼륨이 `available` 상태가 되어야 연결할 수 있다.
- 인스턴스의 연결 가능 볼륨 수를 넘으면 `AttachmentLimitExceeded` 오류가 난다.

### 어느 장치가 그 볼륨인지 확인

출처: [Identify NVMe EBS devices](https://docs.aws.amazon.com/ebs/latest/userguide/identify-nvme-ebs-device.html)

Nitro 인스턴스에서는 `--device /dev/sdf`로 연결해도 OS에서는 `/dev/nvme1n1` 같은 이름으로 보인다. **번호는 부팅할 때마다 바뀔 수 있다.** 그래서 볼륨 ID로 확인한다.

```bash
lsblk -o NAME,SIZE,MOUNTPOINT,SERIAL     # SERIAL 열이 볼륨 ID(vol0123... 형태)
sudo /sbin/ebsnvme-id /dev/nvme1n1       # Amazon Linux
sudo nvme id-ctrl -V /dev/nvme1n1        # 다른 배포판(nvme-cli 필요)
```

Amazon Linux는 udev 규칙으로 `/dev/sdf` → `/dev/nvme1n1` 심볼릭 링크도 만들어 준다.

### 포맷하고 마운트

출처: [Make a volume available for use](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html)

> **데이터 손실 주의**: `mkfs`는 디스크의 기존 데이터를 지운다. 스냅샷에서 복원한 볼륨처럼 이미 데이터가 있는 볼륨에는 실행하지 않는다.

```bash
sudo file -s /dev/nvme1n1        # 결과가 "data"면 파일시스템이 없음(새 볼륨)
lsblk -f                          # FSTYPE 열로도 확인 가능

sudo mkfs -t xfs /dev/nvme1n1    # 새 볼륨일 때만!
sudo mkdir /data
sudo mount /dev/nvme1n1 /data
```

부팅 때 자동으로 마운트하려면 `/etc/fstab`에 **장치 이름 대신 UUID**를 적는다.

```bash
sudo cp /etc/fstab /etc/fstab.orig
sudo blkid /dev/nvme1n1           # UUID 확인
echo 'UUID=aebf131c-6957-451e-8d34-ec978d9581ae  /data  xfs  defaults,nofail  0  2' | sudo tee -a /etc/fstab
sudo umount /data && sudo mount -a   # 설정 검증
```

- `nofail`을 넣으면 볼륨 마운트가 실패해도 인스턴스가 부팅된다.
- AWS 문서는 "Errors in the /etc/fstab file can render a system unbootable."이라고 경고한다. 반드시 `mount -a`로 검증한다.
- 2 TiB보다 큰 볼륨에 파티션을 만들려면 GPT 파티션 테이블이 필요하다.

### 분리(detach)

출처: [Detach a volume](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-detaching-volume.html)

```bash
sudo umount -d /data
aws ec2 detach-volume --volume-id "$VOL"
```

- 마운트된 채로 분리하면 `busy` 상태에서 멈출 수 있다. 루트 볼륨은 인스턴스를 멈춘 뒤에만 분리할 수 있다.
- **Force detach는 최후 수단**이다. 파일시스템 캐시를 내려쓸 기회가 없어 파일시스템이 깨질 수 있으므로, 시도하기 전에 스냅샷을 만들고 분리한 뒤에는 `fsck`로 검사한다.

## Elastic Volumes: 크기·타입·성능 바꾸기

출처: [Modify a volume](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-modify-volume.html), [Monitor modifications](https://docs.aws.amazon.com/ebs/latest/userguide/monitoring-volume-modifications.html)

```bash
# 100 GiB gp2 → 200 GiB gp3, IOPS 6000, 처리량 250 MiB/s
aws ec2 modify-volume --volume-id "$VOL" \
  --volume-type gp3 --size 200 --iops 6000 --throughput 250

aws ec2 describe-volumes-modifications --volume-ids "$VOL" \
  --query 'VolumesModifications[].[ModificationState,Progress]'
```

| 규칙 | 내용 |
| --- | --- |
| 바꿀 수 있는 것 | 크기 늘리기, 성능 올리기·내리기, 타입 변경 |
| **바꿀 수 없는 것** | **크기 줄이기**(더 작은 볼륨을 만들어 `rsync`로 옮겨야 함), 진행 중인 변경 취소 |
| 횟수 | **24시간 동안 최대 4번**. 이전 변경이 `completed`여야 다음 변경 가능 |
| 상태 | `modifying` → `optimizing` → `completed`(드물게 `failed`면 재시도) |
| 성능 | `optimizing` 동안 성능은 기존과 목표 사이. 기존보다 낮아지지는 않음 |
| 소요 시간 | 1 TiB 볼륨은 보통 최대 약 6시간 |
| 비용 | 변경 자체는 무료. 변경을 시작한 시점부터 새 설정 요금 |
| 제약 | 루트 볼륨은 st1·sc1로 바꿀 수 없음. Multi-Attach 볼륨은 io1↔io2만 가능 |

예전에는 "변경 후 6시간 대기"라는 규칙이었지만, 현재 문서는 "24시간에 4번"으로 안내한다.

gp2 → gp3로 바꿀 때 IOPS·처리량을 지정하지 않으면, gp2에서 받던 성능과 gp3 기본값(3,000 IOPS·125 MiB/s) 중 **높은 쪽**으로 설정된다.

### 늘린 크기를 OS에 반영

출처: [Extend the file system after resizing](https://docs.aws.amazon.com/ebs/latest/userguide/recognize-expanded-volume-linux.html)

볼륨이 `optimizing` 상태가 되면 바로 확장할 수 있다. 작업 전에 스냅샷을 만들어 둔다.

```bash
lsblk                                  # 디스크는 커졌지만 파티션은 그대로인지 확인
sudo growpart /dev/nvme0n1 1           # 파티션 1번 확장 (장치와 번호 사이 공백 주의)

df -hT                                 # 파일시스템 종류 확인
sudo xfs_growfs -d /                   # XFS: 마운트 지점을 인자로
sudo resize2fs /dev/nvme0n1p1          # ext4: 장치 이름을 인자로
```

- 파티션이 없는 데이터 볼륨(`/dev/nvme1n1`에 바로 파일시스템을 만든 경우)은 `growpart` 없이 파일시스템만 늘린다.
- LVM을 쓰면 LVM 명령(`pvresize`, `lvextend`)으로 늘린다.

## 암호화

출처: [How EBS encryption works](https://docs.aws.amazon.com/ebs/latest/userguide/how-ebs-encryption-works.html), [Encryption by default](https://docs.aws.amazon.com/ebs/latest/userguide/encryption-by-default.html), [EBS encryption](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption.html)

| 항목 | 내용 |
| --- | --- |
| 암호화되는 것 | 볼륨에 저장된 데이터, **인스턴스↔볼륨 사이 전송 데이터**, 그 볼륨의 모든 스냅샷, 그 스냅샷으로 만든 볼륨 |
| 방식 | AES-256. KMS가 만든 데이터 키를 Nitro 하드웨어 메모리에 둠 |
| 키 | AWS 관리형 `aws/ebs`(리전마다 하나) 또는 대칭 고객 관리형 키 |
| 성능 | IOPS는 같고, 지연에는 "minimal effect" |
| 필요한 KMS 권한 | `kms:CreateGrant`, `Decrypt`, `DescribeKey`, `GenerateDataKeyWithoutPlaintext`, `ReEncrypt*` |

### 리전 기본 암호화 켜기

```bash
aws ec2 enable-ebs-encryption-by-default --region ap-northeast-2
aws ec2 get-ebs-encryption-by-default --region ap-northeast-2
aws ec2 modify-ebs-default-kms-key-id --kms-key-id alias/ebs-default --region ap-northeast-2   # 선택
```

- **리전마다** 켜야 하고, 이미 있는 볼륨·스냅샷에는 적용되지 않는다.
- 조직 전체에 강제하려면 SCP로 암호화되지 않은 볼륨 생성을 막는 방법도 있다([aws-organizations/03](../aws-organizations/03-scp-rcp.md)).

### 바꿀 수 없는 것

- **암호화되지 않은 기존 볼륨·스냅샷을 그 자리에서 암호화할 수 없다.** 스냅샷을 만든 뒤 그 스냅샷으로 암호화된 볼륨을 만들거나, 스냅샷을 암호화해서 복사한다.
- **암호화를 해제할 수 없다.**
- 기존 볼륨의 KMS 키는 바꿀 수 없다. 다른 키로 스냅샷을 복사하면 증분이 아닌 **전체 복사**가 되어 저장 비용이 늘어난다.
- **`aws/ebs` 키로 암호화한 스냅샷은 다른 계정과 공유할 수 없다.** 계정 간 공유가 필요하면 처음부터 고객 관리형 키를 쓴다([04](./04-snapshots.md)).
- KMS 키를 비활성화해도 이미 연결된 볼륨은 계속 동작하지만, **분리 후 다시 연결하는 것은 실패**한다.

## Multi-Attach

출처: [Multi-Attach](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes-multi.html), [NVMe reservations](https://docs.aws.amazon.com/ebs/latest/userguide/nvme-reservations.html)

볼륨 하나를 같은 AZ의 **Nitro 인스턴스 최대 16대**에 동시에 연결한다. 추가 요금은 없다.

| 항목 | 내용 |
| --- | --- |
| 지원 타입 | **io1, io2만**(Windows는 io2만) |
| 파일시스템 | **XFS·ext4 같은 일반 파일시스템은 동시 접근용이 아니다.** 클러스터 파일시스템이 필요하고, 쓰기 순서도 애플리케이션이 관리해야 함 |
| NVMe reservations | 클러스터 소프트웨어가 쓰는 디스크 잠금 기능. io2(2023-09-18 이후 생성)·io1(2026-10-01 이후 생성) Multi-Attach 볼륨에서 기본 사용 가능 |
| 제약 | 부팅 볼륨 불가, 인스턴스 실행 시 콘솔·`RunInstances`로 켤 수 없음, 켜고 끌 때는 분리된 상태여야 함 |
| 장애 영향 | EBS 쪽 장애는 **연결된 모든 인스턴스**에 영향 |

```bash
aws ec2 create-volume --availability-zone ap-northeast-2a \
  --volume-type io2 --size 100 --iops 10000 --multi-attach-enabled
```

일반적인 웹·앱 서버가 파일을 공유하려는 목적이라면 Multi-Attach가 아니라 **EFS**가 맞다([01](./01-overview.md#3단계-다른-저장소와-무엇이-다른가)).

## 모니터링

### CloudWatch 메트릭

출처: [EBS CloudWatch metrics](https://docs.aws.amazon.com/ebs/latest/userguide/using_cloudwatch_ebs.html)

모든 볼륨 타입이 **연결된 동안 1분 간격 메트릭을 무료로** 보낸다(메트릭 기초는 [aws-cloudwatch/02](../aws-cloudwatch/02-metrics.md)).

| 메트릭(`AWS/EBS`) | 뜻 / 보는 법 |
| --- | --- |
| `VolumeReadOps`, `VolumeWriteOps` | I/O 횟수. **IOPS = Sum ÷ period(초)** |
| `VolumeReadBytes`, `VolumeWriteBytes` | 읽고 쓴 바이트 |
| `VolumeQueueLength` | 처리를 기다리는 I/O 수 |
| `VolumeIdleTime` | I/O가 없던 시간 |
| `BurstBalance` | gp2·st1·sc1의 남은 burst 크레딧(%). 0에 가까우면 기본 성능으로 떨어짐 |
| `VolumeThroughputPercentage`, `VolumeConsumedReadWriteOps` | Provisioned IOPS(io1·io2) 볼륨 전용 |
| `VolumeAvgIOPS`, `VolumeAvgThroughput`, `VolumeAvgReadLatency`, `VolumeAvgWriteLatency` | Nitro 인스턴스 전용. 평균 성능·지연 |
| `VolumeIOPSExceededCheck`, `VolumeThroughputExceededCheck` | Nitro 전용. 볼륨 한도를 넘는 요청이 있었으면 1 |
| `VolumeStalledIOCheck` | Nitro 전용. I/O가 멈췄으면 1 |

| 메트릭(`AWS/EC2`, Nitro 인스턴스) | 뜻 |
| --- | --- |
| `InstanceEBSIOPSExceededCheck`, `InstanceEBSThroughputExceededCheck` | **인스턴스** 전체 EBS 한도를 넘었으면 1 |
| `EBSIOBalance%`, `EBSByteBalance%` | 일부 4xlarge 이하 인스턴스의 EBS burst 잔량(24시간에 30분 burst) |

### 초 단위 상세 통계

Nitro 인스턴스에서는 NVMe 장치에서 **최대 1초 단위**의 상세 통계(I/O 수, 지연 히스토그램, 한도 초과 시간 등)를 무료로 볼 수 있다. ([Detailed performance statistics](https://docs.aws.amazon.com/ebs/latest/userguide/nvme-detailed-performance-stats.html))

```bash
sudo nvme amzn stats /dev/nvme1n1    # 2024-11-12 이후 Amazon Linux AMI의 nvme-cli
```

CloudWatch agent나 EBS CSI 드라이버(v1.37.0 이상, Prometheus)로도 수집할 수 있다.

### "느리다"를 진단하는 순서

출처: [I/O characteristics](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-io-characteristics.html)

1. **볼륨 한도에 닿았나?** `VolumeAvgIOPS`·`VolumeAvgThroughput`을 설정값과 비교하고, `Volume*ExceededCheck`가 1인지 본다. 1분 평균에 가려진 순간 폭주(micro-burst)도 여기서 잡힌다.
2. **IOPS보다 처리량이 먼저 막혔나?** I/O 크기가 크면 IOPS 한도 전에 처리량 한도에 닿는다. SSD는 최대 256 KiB, HDD는 1,024 KiB 단위로 I/O를 나누거나 합친다.
3. **gp2·st1·sc1이면 `BurstBalance`**가 바닥났는지 본다.
4. **인스턴스 한도인가?** `InstanceEBS*ExceededCheck`가 1이면 볼륨이 아니라 인스턴스 타입의 EBS 대역폭이 병목이다([02](./02-volume-types.md)).
5. **큐가 부족하거나 넘치나?** 대략 provisioned IOPS 1,000당 평균 큐 길이 1 정도가 기준이다.
6. **스냅샷에서 복원한 볼륨인가?** 처음 읽는 블록은 느리다(→ [04](./04-snapshots.md)의 초기화).

### 볼륨 상태 검사

출처: [Volume status checks](https://docs.aws.amazon.com/ebs/latest/userguide/monitoring-volume-checks.html), [Impaired volumes](https://docs.aws.amazon.com/ebs/latest/userguide/work_volumes_impaired.html)

| 상태 | 뜻 |
| --- | --- |
| `ok` | 정상 |
| `warning` | 성능 저하 등. 스냅샷에서 복원한 볼륨의 초기화 중에는 정상적으로 나올 수 있음 |
| `impaired` | 데이터가 일관되지 않을 수 있어 **EBS가 I/O를 막은 상태** |
| `insufficient-data` | 판단할 데이터 부족 |

```bash
aws ec2 describe-volume-status --filters Name=volume-status.status,Values=impaired
aws ec2 enable-volume-io --volume-id "$VOL"    # I/O 재개 후 fsck로 검사
```

- 검사는 5분마다 한다. I/O 성능 검사는 io1·io2·gp3만 대상이다.
- `--auto-enable-io`로 자동 재개를 켤 수 있지만, AWS는 **부팅 볼륨이나 버려도 되는 볼륨에만** 권장한다.

## 인스턴스 종료 시 볼륨 삭제(DeleteOnTermination)

출처: [Preserve volumes on termination](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/preserving-volumes-on-termination.html)

| 볼륨 | 기본 동작 |
| --- | --- |
| 실행 시 붙은 루트 볼륨 | 인스턴스 종료 시 **삭제** |
| 실행 시 붙은 데이터 볼륨 | 콘솔로 만들면 유지, CLI로 만들면 삭제 |
| 실행 후 연결한 볼륨 | **유지** |

```bash
aws ec2 modify-instance-attribute --instance-id i-0123456789abcdef0 \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"DeleteOnTermination":false}}]'
```

"유지"된 볼륨은 인스턴스가 없어도 **요금이 계속 나간다.** 버려진 볼륨이 쌓이는 가장 흔한 원인이다([05](./05-cost-issues.md)).

## 출처

- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html
- https://docs.aws.amazon.com/cli/latest/reference/ec2/create-volume.html
- https://docs.aws.amazon.com/cli/latest/reference/ec2/attach-volume.html
- https://docs.aws.amazon.com/ebs/latest/userguide/identify-nvme-ebs-device.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-detaching-volume.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-modify-volume.html
- https://docs.aws.amazon.com/ebs/latest/userguide/monitoring-volume-modifications.html
- https://docs.aws.amazon.com/ebs/latest/userguide/recognize-expanded-volume-linux.html
- https://docs.aws.amazon.com/ebs/latest/userguide/how-ebs-encryption-works.html
- https://docs.aws.amazon.com/ebs/latest/userguide/encryption-by-default.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption-requirements.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes-multi.html
- https://docs.aws.amazon.com/ebs/latest/userguide/nvme-reservations.html
- https://docs.aws.amazon.com/ebs/latest/userguide/using_cloudwatch_ebs.html
- https://docs.aws.amazon.com/ebs/latest/userguide/nvme-detailed-performance-stats.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-io-characteristics.html
- https://docs.aws.amazon.com/ebs/latest/userguide/monitoring-volume-checks.html
- https://docs.aws.amazon.com/ebs/latest/userguide/work_volumes_impaired.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/preserving-volumes-on-termination.html

---
*조사 기준일: 2026-10-14. CLI·Linux 명령은 공식 문서 예시를 바탕으로 했고 실제 인스턴스에서 실행하지 않았다. fstab의 UUID는 예시 값이다.*
