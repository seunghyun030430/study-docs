# CLAUDE.md — aws-systems-manager

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: AWS Systems Manager(SSM)를 실무 관점에서 깊게 정리
- 중점 기능: **Session Manager**, **Parameter Store**, **Run Command / Automation**
- 공통 기반(SSM Agent, managed node, IAM, 네트워크)은 `01-overview.md`, `02-architecture.md`에서 다룬다.

## 주제 전용 규칙

- 예제는 별도 `examples/` 없이 **문서 안의 AWS CLI 코드블록**(`bash`)으로 둔다.
- 리소스 ID·계정 ID는 `i-0123456789abcdef0`, `111122223333` 같은 예시 값을 쓴다.
- 1차 출처는 AWS 공식 문서·요금 페이지·CLI 레퍼런스를 우선한다.
- 요금·한도(quota)·기능 제공 여부는 자주 바뀌므로 공식 페이지 링크를 꼭 남긴다.
- 커밋 scope: `docs(aws-systems-manager): ...`
