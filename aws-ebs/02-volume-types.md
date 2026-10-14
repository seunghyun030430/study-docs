# 볼륨 타입과 성능

> 대부분은 **gp3**로 시작한다. 크기와 상관없이 3,000 IOPS·125 MiB/s가 기본으로 포함되고, 2025년부터 최대 64 TiB·80,000 IOPS·2,000 MiB/s까지 따로 올릴 수 있다. 1 ms 미만 지연과 99.999% 내구성이 필요한 핵심 DB는 **io2 Block Express**를 쓴다. 대용량 순차 읽기는 **st1**, 거의 안 읽는 데이터는 **sc1**이다. 실제 성능은 **볼륨 한도와 인스턴스 EBS 한도 중 작은 쪽**으로 정해진다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어(IOPS, 처리량, 지연, 볼륨 타입)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Baseline | 항상 보장되는 기본 성능 |
| Burst / I/O credit | 평소 쌓아 둔 크레딧을 써서 잠깐 기본 성능보다 높게 내는 방식 / 그 크레딧 |
| Provisioned IOPS | 원하는 IOPS를 직접 정하고 그만큼 요금을 내는 방식 |
| I/O 크기 | 한 번의 읽기·쓰기에서 다루는 데이터 크기(예: 16 KiB) |
| Queue depth(큐 길이) | 처리를 기다리며 쌓인 I/O 요청 수 |
| Nitro | AWS의 현세대 인스턴스 하드웨어·하이퍼바이저 플랫폼. 대부분의 최신 인스턴스 타입이 Nitro 기반 |
| EBS-optimized | 인스턴스에 EBS 전용 네트워크 대역폭을 따로 주는 설정 |
| Sequential / random I/O | 연속된 위치를 차례로 읽고 쓰는 I/O / 흩어진 위치를 읽고 쓰는 I/O |

## 타입 비교

출처: [EBS volume types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)

| 타입 | 크기 | 볼륨당 최대 IOPS | 볼륨당 최대 처리량 | 부팅 | Multi-Attach | 주 용도 |
| --- | --- | --- | --- | --- | --- | --- |
| **gp3** | 1 GiB–64 TiB | 80,000 | 2,000 MiB/s | 가능 | 불가 | 대부분의 워크로드 |
| gp2 | 1 GiB–16 TiB | 16,000 | 250 MiB/s | 가능 | 불가 | 이전 세대 범용(신규는 gp3) |
| **io2 Block Express** | 4 GiB–64 TiB | 256,000 | 4,000 MiB/s | 가능 | 가능 | 핵심 DB, 낮은 지연 |
| io1 | 4 GiB–16 TiB | 64,000 | 1,000 MiB/s | 가능 | 가능 | 이전 세대 고성능(신규는 io2) |
| **st1** | 125 GiB–16 TiB | 500 | 500 MiB/s | **불가** | 불가 | 대용량 순차 읽기(로그, 빅데이터, ETL) |
| **sc1** | 125 GiB–16 TiB | 250 | 250 MiB/s | **불가** | 불가 | 거의 접근하지 않는 데이터, 최저 비용 |
| standard(Magnetic) | 1 GiB–1 TiB | 40–200 | 40–90 MiB/s | 가능 | — | 이전 세대(사용 비권장) |

| 타입 | 내구성(연간 고장률) | 성능 달성률 |
| --- | --- | --- |
| io2 | 99.999%(0.001%) | 99.9% 시간 동안 설정 성능 |
| io1 | 99.8–99.9% | 99.9% |
| gp3·gp2 | 99.8–99.9% | 99% |

출처: [General Purpose SSD](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html), [Provisioned IOPS SSD](https://docs.aws.amazon.com/ebs/latest/userguide/provisioned-iops.html)

## 타입별 성능 모델

### gp3 — 성능을 크기와 따로 정한다

출처: [General Purpose SSD](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html), [What's New 2025-09](https://aws.amazon.com/about-aws/whats-new/2025/09/amazon-ebs-size-provisioned-performance-gp3-volumes/)

| 항목 | 값 |
| --- | --- |
| 기본 포함 | **3,000 IOPS, 125 MiB/s**(크기와 무관, 용량 요금에 포함) |
| IOPS 추가 | GiB당 최대 500 IOPS. 80,000 IOPS는 160 GiB 이상에서 가능 |
| 처리량 추가 | IOPS당 최대 0.25 MiB/s. 2,000 MiB/s는 8,000 IOPS 이상에서 가능 |
| Burst | **없음**(항상 설정한 성능) |
| 지연 | 한 자릿수 밀리초 |
| 한도 상향 | 2025-09 발표로 64 TiB·80,000 IOPS·2,000 MiB/s(이전 16 TiB·16,000·1,000). Outposts는 이전 한도 유지 |

```
예: 100 GiB gp3, 6,000 IOPS, 250 MiB/s
 - IOPS 상한: 100 GiB × 500 = 50,000 → 6,000 가능
 - 처리량 상한: 6,000 × 0.25 = 1,500 MiB/s → 250 가능
 - 추가 요금 대상: IOPS 3,000개 초과분, 처리량 125 MiB/s 초과분
```

### gp2 — 크기에 비례하고 burst에 의존한다

| 항목 | 값 |
| --- | --- |
| 기본 IOPS | **GiB당 3 IOPS**(최소 100, 5,334 GiB에서 최대 16,000) |
| Burst | 1 TiB 미만 볼륨은 크레딧을 써서 **3,000 IOPS까지** burst |
| 크레딧 | 최대 540만 개(처음에도 가득 참). GiB당 초당 3개씩 충전. 3,000 IOPS로 최소 30분 burst 가능 |
| 처리량 | 170 GiB 이하 128 MiB/s, 334 GiB 이상 250 MiB/s |
| 확인 | CloudWatch `BurstBalance`(%) |

```
예: 100 GiB gp2
 - 기본: 300 IOPS
 - 크레딧이 있으면 3,000 IOPS까지 burst
 - 크레딧이 바닥나면 300 IOPS로 떨어짐 → "평소엔 빠르다가 갑자기 느려지는" 증상
```

같은 100 GiB라면 **gp3는 항상 3,000 IOPS이면서 20% 더 저렴**하다. gp2를 쓸 이유는 거의 없다(→ [05](./05-cost-issues.md#gp2--gp3-전환)).

### io2 Block Express / io1 — IOPS를 사서 쓴다

출처: [Provisioned IOPS SSD](https://docs.aws.amazon.com/ebs/latest/userguide/provisioned-iops.html)

| 항목 | io2 Block Express | io1 |
| --- | --- | --- |
| IOPS 비율 | GiB당 최대 1,000 IOPS(256 GiB 이상에서 256,000) | GiB당 최대 50 IOPS(100–64,000) |
| 처리량 | IOPS당 0.256 MiB/s, 최대 4,000 MiB/s | 최대 1,000 MiB/s |
| 지연 | Nitro에서 16 KiB I/O 평균 **500마이크로초 미만** | — |
| 제약 | 256,000 IOPS는 Nitro 인스턴스에서만(그 외 32,000) | 64,000 IOPS·1,000 MiB/s는 Nitro에서만 |
| 비고 | 2025-04-30부터 모든 io2 볼륨이 Block Express | 신규는 io2 권장 |

### st1 / sc1 — 처리량 burst 모델

출처: [HDD volumes](https://docs.aws.amazon.com/ebs/latest/userguide/hdd-vols.html)

| 항목 | st1 | sc1 |
| --- | --- | --- |
| 기본 처리량 | TiB당 40 MiB/s | TiB당 12 MiB/s |
| Burst | TiB당 250 MiB/s | TiB당 80 MiB/s |
| 볼륨 최대 | 500 MiB/s | 250 MiB/s |
| 적합 | 큰 블록 순차 읽기 | 하루 몇 번 안 읽는 콜드 데이터 |

HDD는 **작은 랜덤 I/O에 매우 약하다.** 1 MiB보다 작은 요청도 1 MiB 크레딧 하나를 쓰기 때문이다. DB나 부팅 디스크에는 쓰지 않는다.

## 인스턴스 한도: 볼륨보다 먼저 막힐 수 있다

출처: [EBS-optimized instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-optimized.html), [describe-instance-types](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instance-types.html)

> 실제 성능은 "bounded by the instance type's performance limits, or the aggregated performance of its attached volumes, whichever is smaller."

```
볼륨 4개 × gp3 16,000 IOPS = 64,000 IOPS 설정
인스턴스 타입의 EBS 최대 IOPS = 40,000
→ 실제 최대 40,000 IOPS (볼륨 설정 일부가 낭비됨)
```

인스턴스 타입의 EBS 한도는 CLI로 확인한다.

```bash
aws ec2 describe-instance-types --instance-types r6i.large m7g.xlarge \
  --query 'InstanceTypes[].[InstanceType,EbsInfo.EbsOptimizedInfo.BaselineIops,EbsInfo.EbsOptimizedInfo.MaximumIops,EbsInfo.EbsOptimizedInfo.BaselineThroughputInMBps,EbsInfo.EbsOptimizedInfo.MaximumThroughputInMBps]' \
  --output table
```

| 항목 | 내용 |
| --- | --- |
| EBS-optimized | 현세대 인스턴스는 대부분 **기본으로 켜져 있고 끌 수 없다**. 일부 구형은 선택(시간당 추가 요금), 일부는 미지원 |
| Baseline vs Maximum | 작은 인스턴스(일부 4xlarge 이하)는 Maximum까지 **24시간 중 30분만** burst할 수 있고, 나머지 시간은 Baseline |
| 확인 메트릭 | `InstanceEBSIOPSExceededCheck`, `InstanceEBSThroughputExceededCheck`, `EBSIOBalance%`([03](./03-operations.md#모니터링)) |

"볼륨 IOPS를 올렸는데 빨라지지 않는다"면 **인스턴스 한도**부터 확인한다.

## I/O 크기와 IOPS 계산

출처: [I/O characteristics](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-io-characteristics.html)

| 규칙 | 내용 |
| --- | --- |
| 최대 I/O 크기 | SSD 256 KiB, HDD 1,024 KiB. 이보다 큰 I/O는 나뉘어 여러 IOPS로 셈 |
| 작은 순차 I/O | 연속된 작은 I/O는 합쳐서 하나로 셀 수 있음 |
| 최대 IOPS 조건 | 표의 최대 IOPS는 io·gp2 기준 16 KiB, gp3 기준 25.6 KiB I/O에서 측정한 값 |
| 큰 I/O | IOPS 한도보다 **처리량 한도**에 먼저 닿음 |
| 큐 길이 | SSD는 provisioned IOPS 1,000당 평균 큐 길이 약 1을 유지하라고 안내 |

```
예: gp3 3,000 IOPS · 125 MiB/s 볼륨에 256 KiB I/O
 - 처리량 기준 최대: 125 MiB/s ÷ 0.25 MiB ≈ 500 IOPS
 → IOPS 한도(3,000)가 아니라 처리량 한도(125 MiB/s)에서 막힘
```

## 스냅샷에서 만든 볼륨의 초기화

출처: [Initialize volumes](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-initialize.html)

- **새로 만든 빈 볼륨**은 처음부터 최대 성능을 낸다.
- **스냅샷에서 만든 볼륨**은 블록을 처음 읽을 때 S3에서 가져오므로 그 블록의 첫 읽기가 느리다.
- 운영에 투입하기 전에 전체를 한 번 읽어 두면 이 지연을 없앨 수 있다.

```bash
sudo fio --filename=/dev/nvme1n1 --rw=read --bs=1M --iodepth=32 \
  --ioengine=libaio --direct=1 --name=volume-initialize
# 또는
sudo dd if=/dev/nvme1n1 of=/dev/null bs=1M
```

이 명령은 읽기만 하므로 데이터를 바꾸지 않는다. 대신 [Fast Snapshot Restore나 Provisioned Rate for Volume Initialization](./04-snapshots.md)을 쓸 수도 있다.

## 선택 기준

| 상황 | 추천 |
| --- | --- |
| 잘 모르겠다, 일반 웹·앱 서버, 부팅 디스크 | **gp3**(기본값 그대로) |
| 중간 규모 DB, IOPS가 3,000으로 부족 | gp3에 IOPS·처리량 추가 |
| 지연에 민감한 핵심 DB, 99.999% 내구성 필요, Multi-Attach 필요 | **io2 Block Express** |
| Kafka·로그·빅데이터처럼 큰 파일을 순차로 읽음, 용량이 큼 | **st1** |
| 아카이브성 데이터를 가끔 읽음 | **sc1**(또는 S3를 먼저 검토) |
| gp2·io1을 쓰고 있음 | gp3·io2로 전환 검토(Elastic Volumes로 무중단) |

## 이슈 / 주의

- **gp2의 burst 함정**: 작은 gp2 볼륨은 평소 크레딧으로 빠르다가, 배치 작업 같은 장시간 부하에서 크레딧이 떨어지면 갑자기 느려진다. `BurstBalance` 알람을 걸거나 gp3로 바꾼다.
- **인스턴스 한도 무시**: 볼륨 성능만 올리고 인스턴스 타입을 그대로 두면 돈만 더 낸다.
- **작은 인스턴스의 30분 burst**: 테스트할 때는 빨랐는데 운영에서 오래 돌리면 느려지는 원인이 될 수 있다.
- **HDD에 랜덤 I/O**: st1·sc1에 DB를 올리면 매우 느리다.
- **복원 직후 성능 측정**: 스냅샷에서 만든 볼륨은 초기화 전까지 느리므로, 벤치마크는 초기화 후에 한다.

## 출처

- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html
- https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html
- https://docs.aws.amazon.com/ebs/latest/userguide/provisioned-iops.html
- https://docs.aws.amazon.com/ebs/latest/userguide/hdd-vols.html
- https://aws.amazon.com/about-aws/whats-new/2025/09/amazon-ebs-size-provisioned-performance-gp3-volumes/
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-optimized.html
- https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instance-types.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-io-characteristics.html
- https://docs.aws.amazon.com/ebs/latest/userguide/ebs-initialize.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/RootDeviceStorage.html

---
*조사 기준일: 2026-10-14. 성능 한도는 자주 바뀌므로 적용 전에 볼륨 타입 문서를 다시 확인할 것. 계산 예시는 이 레포에서 만든 것이다.*
