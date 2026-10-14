# AWS CloudFormation 정리

AWS CloudFormation을 실무 관점에서 정리한 문서 모음. 중점은 **템플릿 문법 / 스택 운영 / StackSets·멀티 계정 / CDK·Terraform 비교 / 검증 도구·흔한 실패**. 조사 기준일 **2026-10-14**.

관련 주제: 계정·OU는 [aws-organizations](../aws-organizations/README.md), IAM은 [aws-iam](../aws-iam/README.md), Quick Setup의 StackSets 사용은 [aws-systems-manager/06](../aws-systems-manager/06-quick-setup.md).

## 한눈에 보기

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| CloudFormation이 뭔가? | 템플릿(YAML/JSON)대로 AWS 리소스를 만들고 고치고 지우는 AWS 기본 IaC. `AWS::*` 리소스는 무료 | [01](./01-overview.md) |
| 템플릿의 핵심은? | 필수는 `Resources`. 값은 `!Ref`·`!GetAtt`·`!Sub`로 조합, 비밀은 동적 참조 | [02](./02-templates.md) |
| 스택끼리 값을 넘기려면? | 같은 수명이면 nested stack, 다른 팀·수명이면 Export/ImportValue, 계정·리전을 넘으면 `Fn::GetStackOutput` | [02](./02-templates.md) |
| 업데이트 전에 꼭 할 일? | **change set으로 Replacement 확인**. 데이터 리소스엔 `DeletionPolicy`+`UpdateReplacePolicy` | [03](./03-stack-operations.md) |
| 실패하면? | 자동 rollback. 생성 실패(`ROLLBACK_COMPLETE`)는 삭제만 가능, `UPDATE_ROLLBACK_FAILED`는 continue-update-rollback | [03](./03-stack-operations.md) |
| 콘솔에서 바뀐 걸 찾으려면? | drift detection, 되돌리려면 drift-aware change set | [03](./03-stack-operations.md) |
| 조직 전체 계정에 배포하려면? | StackSets(service-managed) + auto-deployment. management 계정에는 배포 안 됨 | [04](./04-stacksets.md) |
| CDK vs Terraform? | CDK = 코드로 CloudFormation 템플릿 생성(자동 rollback). Terraform = 별도 도구, state 직접 관리, 자동 rollback 없음, 멀티 클라우드 | [05](./05-cdk-terraform.md) |
| 배포 전 검사는? | cfn-lint(문법·속성) → Guard(조직 규칙) → Hooks(계정 강제) | [06](./06-issues.md) |

| 상태 | 뜻 | 할 수 있는 것 |
| --- | --- | --- |
| `ROLLBACK_COMPLETE` | 생성 실패 후 되돌림 | **삭제만** |
| `UPDATE_ROLLBACK_COMPLETE` | 업데이트 실패 후 이전 상태로 복구 | 다시 업데이트 |
| `UPDATE_ROLLBACK_FAILED` | 되돌리기도 실패 | 원인 수정 후 `continue-update-rollback` |
| `DELETE_FAILED` | 삭제 실패(비어 있지 않은 버킷 등) | 원인 정리 후 재삭제, `--retain-resources` |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요(단계별 입문) — IaC, 템플릿·스택·논리/물리 ID, 배포 흐름과 상태, StackSets 소개, CDK·SAM·IaC generator·Terraform, 요금, 용어 정리 |
| [02-templates.md](./02-templates.md) | 섹션 구성, Parameters(AWS·SSM 타입), 내장 함수·의사 파라미터, Conditions·Mappings, LanguageExtensions(ForEach), 의존성, Export·nested stack·GetStackOutput, 동적 참조, Transform·custom resource |
| [03-stack-operations.md](./03-stack-operations.md) | 업데이트 동작(수정 vs 교체), change set·drift-aware change set, rollback·rollback trigger·UPDATE_ROLLBACK_FAILED, drift detection, DeletionPolicy·UpdateReplacePolicy·termination protection·stack policy, import·IaC generator·stack refactoring, service role |
| [04-stacksets.md](./04-stacksets.md) | 구조, self-managed vs service-managed, auto-deployment, 동시성·실패 허용, 배포 예시, 상태 읽기, 한도 |
| [05-cdk-terraform.md](./05-cdk-terraform.md) | CDK construct 수준·명령·주의점, Terraform 기본(state·S3 잠금·rollback 없음·라이선스), 비교표, 선택 기준 |
| [06-issues.md](./06-issues.md) | 검증 도구(validate-template, 사전 검증, cfn-lint, Guard, Hooks), 흔한 실패 표, 한도, 안티패턴 |

## 출처 (1차)

- CloudFormation User Guide: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html
- Template Reference: https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/introduction.html
- AWS CLI `cloudformation`: https://docs.aws.amazon.com/cli/latest/reference/cloudformation/
- 한도: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html
- 요금: https://aws.amazon.com/cloudformation/pricing/
- CDK Developer Guide: https://docs.aws.amazon.com/cdk/v2/guide/home.html
- Terraform: https://developer.hashicorp.com/terraform/intro
- cfn-lint: https://github.com/aws-cloudformation/cfn-lint

---
*이 문서는 학습·정리 목적의 요약이다. 템플릿 예시는 cfn-lint 1.57.2로 검사했지만 실제 계정에 배포하지는 않았다. 스택 삭제·리소스 교체는 데이터를 지울 수 있으므로 change set으로 먼저 확인할 것.*
