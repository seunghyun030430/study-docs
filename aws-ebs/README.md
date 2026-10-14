# Amazon EBS 정리

Amazon EBS(Elastic Block Store)를 실무 관점에서 정리한 문서 모음. 중점은 **볼륨 타입·성능 / 운영(연결·변경·암호화·모니터링) / 스냅샷·백업 / 비용·이슈**. 조사 기준일 **2026-10-14**.

관련 주제: 메트릭은 [aws-cloudwatch](../aws-cloudwatch/README.md), KMS·IAM은 [aws-iam](../aws-iam/README.md).

## 한눈에 보기

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| EBS가 뭔가? | EC2에 붙이는 블록 스토리지(가상 디스크). 인스턴스와 독립, **AZ에 묶임** | [01](./01-overview.md) |
| 어떤 타입을 쓰나? | 기본은 **gp3**. 핵심 DB는 io2 Block Express, 대용량 순차 읽기는 st1, 콜드 데이터는 sc1 | [02](./02-volume-types.md) |
| 성능이 안 나오면? | 볼륨 한도 → 처리량 한도(I/O 크기) → burst 크레딧 → **인스턴스 EBS 한도** 순으로 확인 | [02](./02-volume-types.md), [03](./03-operations.md) |
| 크기를 바꾸려면? | Elastic Volumes로 무중단 변경. **줄일 수 없고 24시간에 4번까지**. 늘린 뒤 파티션·파일시스템 확장 필요 | [03](./03-operations.md) |
| 암호화는? | 리전별 **기본 암호화** 켜기. 계정 간 공유가 필요하면 고객 관리형 KMS 키 | [03](./03-operations.md) |
| 백업은? | 스냅샷(증분). 자동화는 DLM(무료, EBS 전용) 또는 AWS Backup(여러 서비스·조직) | [04](./04-snapshots.md) |
| 복원한 볼륨이 느리면? | 첫 읽기 지연(lazy loading). fio 초기화, FSR, 초기화 속도 지정 | [04](./04-snapshots.md) |
| 비용을 줄이려면? | gp2→gp3(20% 저렴), 연결 안 된 볼륨·남은 스냅샷 정리, Compute Optimizer | [05](./05-cost-issues.md) |

| 타입 | 최대 IOPS / 처리량 | 용량 단가(us-east-1) | 한 줄 |
| --- | --- | --- | --- |
| gp3 | 80,000 / 2,000 MiB/s | GB-월 $0.08 + 추가 성능 | 기본 선택, 3,000 IOPS·125 MiB/s 포함 |
| gp2 | 16,000 / 250 MiB/s | GB-월 $0.10 | 크기 비례 + burst, gp3로 전환 권장 |
| io2 Block Express | 256,000 / 4,000 MiB/s | GB-월 $0.125 + IOPS-월 $0.065부터 | 1 ms 미만 지연, 99.999% 내구성 |
| st1 | 500 / 500 MiB/s | GB-월 $0.045 | 순차 처리량, 부팅 불가 |
| sc1 | 250 / 250 MiB/s | GB-월 $0.015 | 최저가 콜드, 부팅 불가 |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요(단계별 입문) — 블록 스토리지, EBS의 성질(AZ·내구성), instance store·EFS·S3 비교, 타입·스냅샷·암호화 소개, 요금 개요, 용어 정리 |
| [02-volume-types.md](./02-volume-types.md) | 타입 비교표, gp3·gp2·io2·io1·st1·sc1 성능 모델, 인스턴스 EBS 한도, I/O 크기와 IOPS 계산, 초기화, 선택 기준 |
| [03-operations.md](./03-operations.md) | 생성·연결·NVMe 장치 확인·마운트·fstab, Elastic Volumes와 파일시스템 확장, 암호화, Multi-Attach, CloudWatch 메트릭과 성능 진단, 상태 검사, DeleteOnTermination |
| [04-snapshots.md](./04-snapshots.md) | 증분 저장 원리, 일관성, 복원과 초기화(FSR·초기화 속도), 복사·공유·Block public access, Archive tier, Recycle Bin·snapshot lock, DLM vs AWS Backup |
| [05-cost-issues.md](./05-cost-issues.md) | 요금 구조(us-east-1·서울), 계산 예시, gp2→gp3 전환, 버려진 리소스 찾기, 한도, 흔한 장애 표, 안티패턴 |

## 출처 (1차)

- Amazon EBS User Guide: https://docs.aws.amazon.com/ebs/latest/userguide/what-is-ebs.html
- 볼륨 타입: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html
- EBS-optimized 인스턴스: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-optimized.html
- 요금: https://aws.amazon.com/ebs/pricing/
- 한도: https://docs.aws.amazon.com/general/latest/gr/ebs-service.html
- AWS Backup: https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html

---
*이 문서는 학습·정리 목적의 요약이다. 성능 한도와 요금은 자주 바뀌므로 적용 전에 공식 문서를 다시 확인할 것. 디스크를 포맷·삭제하는 명령은 데이터를 지우므로 실행 전에 대상을 반드시 확인할 것.*
