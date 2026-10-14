# CLAUDE.md — aws-ebs

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: Amazon EBS(Elastic Block Store)를 실무 관점에서 정리
- 중점: 볼륨 타입·성능 / 운영(연결·변경·암호화·Multi-Attach·모니터링) / 스냅샷·백업 / 비용·이슈
- 관련 주제: 메트릭은 [`aws-cloudwatch`](../aws-cloudwatch/README.md), KMS·IAM 용어는 [`aws-iam`](../aws-iam/README.md)을 링크하고 반복 설명하지 않는다.
- 예제 스택: 문서 안의 AWS CLI(`aws ec2`, `aws dlm`) + Linux 명령(`lsblk`, `growpart`, `xfs_growfs`, `resize2fs`, `fio` 등) 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- 1차 출처는 Amazon EBS User Guide, EC2 User Guide, AWS CLI 레퍼런스, 요금 페이지(또는 Price List API)를 우선한다.
- **성능 수치(IOPS·처리량)와 요금은 자주 바뀐다.** 조사 기준일과 출처를 꼭 남기고, 금액은 기준 리전(us-east-1)과 서울 값을 구분한다.
- 단위는 AWS 표기를 따른다(GiB, MiB/s, IOPS).
- 디스크를 지우거나 포맷하는 명령(`mkfs`, 볼륨 삭제 등)에는 데이터 손실 경고를 함께 적는다.
- 커밋 scope: `docs(aws-ebs): ...`
