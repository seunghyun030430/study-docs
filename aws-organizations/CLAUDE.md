# CLAUDE.md — aws-organizations

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: AWS Organizations로 여러 AWS 계정을 묶어 관리하는 방법을 실무 관점에서 정리
- 중점: 조직 구조·OU 설계 / SCP·RCP 설계 / IAM Identity Center / 기타 관리 정책·통합 서비스(root 중앙 관리, Control Tower)
- 선행 주제: [`aws-iam`](../aws-iam/README.md). IAM 기본 용어(principal, role, 정책 평가 로직 등)는 그쪽 문서를 링크하고 반복 설명하지 않는다.
- 예제 스택: 문서 안의 JSON 정책 + AWS CLI(`aws organizations`, `aws sso-admin`, `aws configure sso`) 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- 1차 출처는 Organizations User Guide, IAM Identity Center User Guide, AWS 멀티 계정 백서, AWS 공식 예시 저장소(aws-samples)를 우선한다.
- SCP·RCP 예시는 **그대로 붙여 넣으면 계정 전체가 막힐 수 있다**는 경고와 함께 쓰고, 예외 처리(관리 role 제외 등)와 테스트 방법을 같이 적는다.
- 정책 JSON은 ` ```json ` 코드펜스, 계정 ID는 `111122223333`, 조직 ID는 `o-exampleorgid`, OU ID는 `ou-exam-ple12345` 같은 예시 값을 쓴다.
- 커밋 scope: `docs(aws-organizations): ...`
