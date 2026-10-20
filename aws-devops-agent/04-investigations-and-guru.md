# CloudWatch Investigations와 DevOps Guru

> **CloudWatch Investigations**는 CloudWatch 안의 무료 생성형 AI 조사 기능이다. **알람 액션으로 바로 조사를 시작**할 수 있고, 메트릭·로그·X-Ray·CloudTrail 변경을 묶어 가설을 내놓으며, SSM Automation runbook을 제안한다. 다만 계정당 조사 그룹 1개, 동시 조사 2개, 월 150회의 AI 분석 한도가 있고, **서울 리전은 정식 출시 목록에 없다.** **DevOps Guru**는 2026-10-29부터 신규 가입을 받지 않고 2027-09-30에 끝나므로, 쓰고 있다면 2027-06-30까지 CloudWatch·DevOps Agent로 옮긴다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어와 [aws-cloudwatch/03](../aws-cloudwatch/03-alarms.md)의 알람·composite alarm·anomaly detection을 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Investigation group | CloudWatch Investigations 설정 단위(계정당 1개). 권한·보존 기간·연동을 정함 |
| Enhanced investigation | AI 분석이 들어간 조사(월 150회 한도) |
| Cross-account observability | 모니터링 계정 하나에서 여러 계정의 텔레메트리를 보는 CloudWatch 기능 |
| Insight(DevOps Guru) | 관련 이상(anomaly)들을 묶고 권장 사항을 붙인 결과 |
| Reactive / proactive insight | 지금 일어나는 문제 / 일어날 것으로 예측되는 문제 |
| Resource collection | DevOps Guru가 분석할 리소스 범위(계정 전체, CloudFormation 스택, 태그) |

## CloudWatch Investigations

### 무엇이고 언제 나왔나

출처: [Investigations](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations.html), [GA 발표](https://aws.amazon.com/about-aws/whats-new/2025/06/ga-accelerate-troubleshooting-amazon-cloudwatch-investigations)

| 날짜 | 내용 |
| --- | --- |
| 2024-12-03 | "Amazon Q Developer operational investigations" 이름으로 미리 보기(us-east-1) |
| 2025-06-24 | **CloudWatch investigations로 정식 출시, 추가 요금 없음** |

정식 출시 리전에는 도쿄·싱가포르·시드니 등이 있지만 **서울은 없다.** 현재 문서의 리전 간 추론 표에도 서울이 보이지 않는다(확인 필요). 아시아 리전의 조사는 미국 리전에서 추론될 수 있다(데이터는 기본 리전에만 저장).

### 두 가지 사용 방식

| 항목 | 설정 없이 바로 | **Investigation group 설정 후** |
| --- | --- | --- |
| 권한 | 로그인한 사용자의 권한 | 전용 IAM role(`aiops.amazonaws.com`이 맡음) |
| 보존 | 사용자 세션에 묶이고 **24시간 뒤 삭제** | 7–90일(수동으로 끝내지 않으면 7일 뒤 자동 종료) |
| 기능 | 한 번 실행 | 제안 수락·버리기, 메모, **runbook 실행**, 팀 공유, 계정 간, 보고서 생성 |
| 백그라운드 | 없음 | role이 주기적으로 리소스를 스캔해 topology 파악 |

한도: **계정당 investigation group 1개, 동시 진행 조사 2개, AI 분석 조사 월 150회**.

### 조사 시작: 알람 액션

출처: [Configure alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-configure-alarms.html), [Create investigation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-CreateInvestigation.html)

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name checkout-5xx \
  --namespace AWS/ApplicationELB --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/my-alb/50dc6c495c0c9188 \
  --statistic Sum --period 60 --evaluation-periods 5 --datapoints-to-alarm 3 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions \
    "arn:aws:aiops:ap-northeast-1:111122223333:investigation-group/abc123#DEDUPE_STRING=checkout" \
    arn:aws:sns:ap-northeast-1:111122223333:ops-alerts
```

- 알람 액션 ARN 형식은 `arn:aws:aiops:<region>:<account>:investigation-group/<id>`이다. `#DEDUPE_STRING=값`을 붙이면 같은 값을 가진 알람들이 **조사 하나로 합쳐진다.**
- 그 밖의 시작 방법: 콘솔(80개 이상 콘솔)의 **Investigate** 버튼, Amazon Q 채팅, Application Signals SLO.
- 위 예시는 지원 리전인 도쿄(ap-northeast-1)를 썼다. 알람 액션의 원리는 [aws-cloudwatch/03](../aws-cloudwatch/03-alarms.md#액션)을 참고한다.

### 무엇을 보고, 무엇을 제안하나

| 입력 데이터 | 출력 |
| --- | --- |
| CloudWatch 메트릭·로그(Logs Insights 쿼리 포함), X-Ray 트레이스, 배포 이벤트, AWS Health 이벤트, **CloudTrail 변경 이벤트**, (선택) Application Signals, EKS, 애플리케이션 경계 태그 | 관련 메트릭·로그, **원인 가설**, 문서·re:Post 글, **SSM Automation runbook 제안** |

- runbook은 **실행하는 운영자의 IAM 권한**으로 돈다. 실행 결과와 SSM 요금은 사용자 책임이다([aws-systems-manager/05](../aws-systems-manager/05-run-command-automation.md)).
- 사람이 각 제안을 **수락하거나 버리며** 조사를 좁혀 간다.

### IAM, 암호화, 계정 간, 채팅

출처: [Get started](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-GetStarted-Group.html), [Security](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-Security.html), [Cross-account](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-cross-account.html), [Integrations](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-Integrations.html)

| 항목 | 내용 |
| --- | --- |
| 그룹 생성 권한 | `AIOpsConsoleAdminPolicy` 또는 AdministratorAccess |
| 조사 role | `aiops.amazonaws.com`을 신뢰, `AIOpsAssistantPolicy`. trust policy에 `aws:SourceAccount`·`aws:SourceArn` 조건 권장 |
| 사용자 정책 | `AIOpsOperatorAccess`, `AIOpsReadOnlyAccess` |
| 암호화 | 기본 AWS 소유 키, 선택적으로 고객 관리형 KMS 키 |
| 계정 간 | CloudWatch cross-account observability 기반, 모니터링 계정 + 소스 계정 최대 25개 |
| 채팅 | Slack·Teams를 SNS 경유로 연동. SNS 토픽 정책에서 `aiops.amazonaws.com`의 `sns:Publish` 허용 |

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "aiops.amazonaws.com" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "aws:SourceAccount": "111122223333" },
        "ArnLike": { "aws:SourceArn": "arn:aws:aiops:ap-northeast-1:111122223333:*" }
      }
    }
  ]
}
```

위 trust policy는 문서가 권하는 조건 키로 이 레포에서 작성한 예시다. `aws:SourceArn`의 정확한 형식은 공식 예시와 대조하지 못했다(확인 필요). confused deputy 방지 원리는 [aws-iam/04](../aws-iam/04-roles-sts.md#confused-deputy와-external-id)를 참고한다.

### DevOps Agent와의 관계

- DevOps Guru 이전 안내는 두 서비스를 **나란히 대안으로** 소개한다. DevOps Agent는 "AI-assisted investigation"을, CloudWatch Investigations는 "the same type of investigation from within the CloudWatch console"을 제공한다는 설명이다.
- 2026-09-23 정식 출시된 **CloudWatch Omni**(자연어 관측 기능)는 원인 분석에 **DevOps Agent를 내장 에이전트로** 쓴다. 이를 쓰려면 DevOps Agent Space를 연결해야 한다. ([Omni GA](https://aws.amazon.com/about-aws/whats-new/2026/09/amazon-cloudwatch-omni-ai/))
- CloudWatch Investigations가 없어지거나 DevOps Agent로 대체된다는 공지는 찾지 못했다(2026-10-20 기준).

## Amazon DevOps Guru

### 지원 종료 일정

출처: [End of support](https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-end-of-support.html), [Transition](https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-transition.html), [Service availability update](https://aws.amazon.com/about-aws/whats-new/2026/09/aws-service-availability/)

> "Amazon DevOps Guru will stop accepting new customers on October 29, 2026. Current customers will not be impacted and can continue using Amazon DevOps Guru until September 30, 2027."

| 날짜 | 일어나는 일 |
| --- | --- |
| 2026-09-29 | AWS 서비스 가용성 업데이트에서 발표 |
| **2026-10-29** | 신규 고객 가입 중단(그 전에 가입한 계정은 계속 사용) |
| 그때까지 | 보안 패치·치명적 버그 수정·Support만, **새 기능 없음** |
| **2027-06-30** | AWS 권장 이전 완료 시점(새 구성과 나란히 비교할 시간 확보) |
| **2027-09-30** | **지원 종료**: 콘솔·API 중지, 과거 인사이트 조회 불가, CloudFormation 리소스 타입 제거(스택 생성·업데이트 실패) |

### 무엇을 하던 서비스인가

출처: [Concepts](https://docs.aws.amazon.com/devops-guru/latest/userguide/concepts.html), [Coverage](https://docs.aws.amazon.com/devops-guru/latest/userguide/choose-coverage.html), [Pricing](https://aws.amazon.com/devops-guru/pricing/)

| 항목 | 내용 |
| --- | --- |
| 분석 데이터 | CloudWatch 메트릭·로그 그룹·이벤트, 관련 CloudTrail, CodeDeploy 배포 이벤트 |
| 결과 | anomaly(이상) → **insight**(묶음 + 권장 사항). reactive(지금) / proactive(예측). 심각도 high·medium·low |
| DevOps Guru for RDS | Performance Insights의 `DBLoad` 분석. 상세 분석은 Aurora MySQL·Aurora PostgreSQL·RDS for PostgreSQL만 |
| 범위 | 계정 전체 / CloudFormation 스택 / 태그(키는 `Devops-guru-` 접두사) / None(끄기) |
| 연동 | SNS(최대 2개), OpsCenter OpsItem, EventBridge(`aws.devops-guru`), Organizations 위임 관리자 |
| 요금 | 분석하는 **활성 리소스-시간**당 Group A(Lambda·S3) $0.0028, Group B(그 외) $0.0042, API 호출 1만 건당 $0.40 |

### 이전 계획

| 단계 | 할 일 |
| --- | --- |
| 1 | **IaC 정리가 최우선**: CloudFormation·CDK·Terraform에서 DevOps Guru 리소스 제거(지원 종료 후 스택 업데이트가 실패함). Terraform은 `aws_devopsguru_resource_collection` 등 |
| 2 | 메트릭 이상 탐지 → CloudWatch anomaly detection 알람 + composite alarm |
| 3 | RDS → CloudWatch Database Insights + `DBLoad` 이상 탐지 알람 |
| 4 | 로그 이상 → CloudWatch Logs anomaly detection([aws-cloudwatch/04](../aws-cloudwatch/04-logs.md#그-밖의-기능)) |
| 5 | 조사·원인 연결 → DevOps Agent 또는 CloudWatch Investigations |
| 6 | proactive 인사이트 → **대체 없음**. 사용률 임계 알람, Service Quotas 사용량 알람으로 보완 |
| 7 | 기존 인사이트를 API로 내보내고, 새 인사이트는 SNS로 받아 보관 |
| 8 | 범위를 **None**으로 바꿔 과금 중지 |

```bash
# 현재 상태 확인
aws devops-guru describe-account-health
aws devops-guru list-insights --status-filter '{"Ongoing":{"Type":"REACTIVE"}}'

# CloudFormation 스택 기준 범위에서 스택 제거 (이전 후 정리 단계)
aws devops-guru update-resource-collection --action REMOVE \
  --resource-collection '{"CloudFormation":{"StackNames":["my-stack"]}}'
```

- `update-resource-collection`의 스택 수 한도는 CLI 레퍼런스(500)와 콘솔·한도 문서(1,000)가 다르다(확인 필요).
- `list-insights`의 `--status-filter` 값은 문서의 옵션 구조로 이 레포에서 작성했다.
- 범위를 None으로 바꿔도 과거 인사이트를 조회하면 API 요금이 조금 나올 수 있다.

## 이슈 / 주의

- **CloudWatch Investigations의 서울 지원 여부**를 먼저 확인한다. 지원하지 않으면 서울 워크로드는 DevOps Agent를 검토한다.
- **월 150회 AI 분석 한도**가 있다. 모든 알람을 조사로 연결하면 금방 소진된다. 중요한 알람만 연결하고 `DEDUPE_STRING`으로 묶는다.
- **runbook 제안은 그대로 실행하지 않는다.** 실행자의 권한으로 실제 변경이 일어난다.
- **DevOps Guru는 신규로 도입하지 않는다.** 이미 쓰고 있다면 IaC 정리부터 시작한다.
- DevOps Guru의 **proactive 인사이트는 대체 기능이 없다.** 예측 알림이 중요했다면 사용량·한도 알람을 따로 설계한다.

## 출처

- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations.html
- https://aws.amazon.com/about-aws/whats-new/2024/12/amazon-q-developer-operational-investigation-preview/
- https://aws.amazon.com/about-aws/whats-new/2025/06/ga-accelerate-troubleshooting-amazon-cloudwatch-investigations
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-Retention.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-configure-alarms.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-CreateInvestigation.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-GetStarted-Group.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/suggested-investigation-actions.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-Security.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-cross-account.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations-Integrations.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/investigations-costs.html
- https://aws.amazon.com/about-aws/whats-new/2026/09/amazon-cloudwatch-omni-ai/
- https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-end-of-support.html
- https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-transition.html
- https://aws.amazon.com/about-aws/whats-new/2026/09/aws-service-availability/
- https://docs.aws.amazon.com/devops-guru/latest/userguide/concepts.html
- https://docs.aws.amazon.com/devops-guru/latest/userguide/choose-coverage.html
- https://aws.amazon.com/devops-guru/pricing/
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/devops-guru/update-resource-collection.html

---
*조사 기준일: 2026-10-20. CLI·JSON 예시는 실제 계정에서 실행하지 않았다.*
