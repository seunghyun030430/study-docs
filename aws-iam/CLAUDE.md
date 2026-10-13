# CLAUDE.md — aws-iam

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: AWS IAM을 실무 관점에서 깊게 정리
- 중점: 정책 평가 로직 / Role·STS·교차 계정 / 워크로드 자격증명 / 권한 점검·최소 권한
- 범위 밖: AWS Organizations·IAM Identity Center 상세(→ [`aws-organizations`](../aws-organizations/README.md)). SCP·RCP는 정책 평가 로직 안에서만 다룬다.
- 예제 스택: 문서 안의 JSON 정책 + AWS CLI(`aws iam`, `aws sts`, `aws accessanalyzer`) 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- 1차 출처는 IAM User Guide·STS API Reference·Service Authorization Reference·AWS CLI 레퍼런스를 우선한다.
- 정책 JSON은 ` ```json ` 코드펜스로 쓰고 `"Version": "2012-10-17"`을 포함한다.
- 계정 ID·ARN은 `111122223333`(내 계정), `444455556666`(다른 계정) 같은 예시 값을 쓴다.
- 정책 예시는 "그대로 붙여 넣으면 동작하는 최소 예시"로 쓰고, 운영에 쓸 때 좁혀야 할 부분(Resource, Condition)을 함께 적는다.
- 한도(quota)·요금은 자주 바뀌므로 공식 페이지 링크와 조사 기준일을 남긴다.
- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다. 새 문서·새 용어를 추가할 때 이 표도 함께 갱신한다.
- 커밋 scope: `docs(aws-iam): ...`
