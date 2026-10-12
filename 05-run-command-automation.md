# Run Command / Automation

> **Run Command**는 "노드 안에서 명령을 실행"하고(Command document), **Automation**은 "AWS API 호출과 노드 명령을 여러 단계로 엮은 런북을 실행"한다(Automation runbook). 둘 다 대상 지정(태그·리소스 그룹)과 rate control(`max-concurrency`/`max-errors`)을 지원한다. 단, Run Command의 `Success`는 **종료 코드 0**이라는 뜻일 뿐이다.

## 개요 / 비교

| 항목 | Run Command | Automation |
| --- | --- | --- |
| 실행 위치 | 노드 내부(SSM Agent가 실행) | Systems Manager 서비스(필요 시 `aws:runCommand`로 노드에 위임) |
| 문서 타입 | Command(schema 2.2 권장) | Automation(schema 0.3) |
| 대표 문서 | `AWS-RunShellScript`, `AWS-RunPowerShellScript` | `AWS-RestartEC2Instance`, `AWS-StopEC2Instance`, `AWS-UpdateLinuxAmi` |
| 단계 | 문서 내 plugin(step) 순차 실행 | 23개 action(`aws:executeAwsApi`, `aws:branch`, `aws:approve` 등)으로 분기·대기·승인 |
| 권한 | 호출자의 `ssm:SendCommand` + 노드의 instance profile | 호출자 권한 또는 **assume role**(`AutomationAssumeRole`) |
| 요금 | EC2 무료, 하이브리드 호출당 $0.002(2026-09-30~) | step당 $0.002, `executeScript`는 초당 $0.00003 추가 |
| 트리거 | 콘솔/CLI, EventBridge, State Manager, Maintenance Windows | 콘솔/CLI, EventBridge, Maintenance Windows, State Manager |

출처: [Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command.html), [Automation](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html), [Pricing](https://aws.amazon.com/systems-manager/pricing/)

## Run Command

### 대상 지정

출처: [Run commands at scale](https://docs.aws.amazon.com/systems-manager/latest/userguide/send-commands-multiple.html), [send-command CLI](https://docs.aws.amazon.com/cli/latest/reference/ssm/send-command.html)

| 방식 | 형식 | 제한 |
| --- | --- | --- |
| 인스턴스 ID | `--instance-ids i-a i-b` 또는 `Key=instanceids,Values=...` | `--instance-ids` 최대 50개 |
| 태그 | `--targets Key=tag:Env,Values=prod` | 키 최대 5개, 키당 값 5개, 여러 키는 AND |
| 리소스 그룹 | `Key=resource-groups:Name,Values=my-group` | 명령당 그룹 1개 |

`--targets`는 항목을 최대 5개까지 받는다. 대규모 플릿은 인스턴스 ID 나열보다 태그나 리소스 그룹으로 지정하는 것이 권장된다.

### Rate control

| 옵션 | 기본값 | 동작 |
| --- | --- | --- |
| `--max-concurrency` | 50 | 동시에 실행할 노드 수(숫자 또는 %). 1 → 2 → … 식으로 지수적으로 늘린다 |
| `--max-errors` | 0 | 이 값을 **넘는** 오류가 나면 새 노드로의 전송을 중단. 예: 3이면 4번째 오류에서 중단, 50대 중 10%면 6번째 오류에서 중단 |

이미 실행 중인 호출은 중단되지 않고 끝까지 실행된다. 오류 한도를 엄격히 지키려면 `max-concurrency 1`을 써야 한다. (같은 출처)

### 출력과 상태

- `get-command-invocation`의 `StandardOutputContent`는 앞 **24,000자**만, `list-command-invocations --details`의 plugin output은 **2,500자**만 담는다. 전체 출력이 필요하면 S3나 CloudWatch Logs로 보낸다. ([get-command-invocation](https://docs.aws.amazon.com/cli/latest/reference/ssm/get-command-invocation.html), [CommandPlugin](https://docs.aws.amazon.com/systems-manager/latest/APIReference/API_CommandPlugin.html))
- S3에 출력을 쓸 때는 명령을 보낸 사용자가 아니라 **노드의 instance profile 권한**을 쓴다. ([Running commands](https://docs.aws.amazon.com/systems-manager/latest/userguide/running-commands-console.html))
- CloudWatch 출력에서 로그 그룹을 지정하지 않으면 기본 이름은 `aws/ssm/<DocumentName>`이다.

상태 값([Monitor commands](https://docs.aws.amazon.com/systems-manager/latest/userguide/monitor-commands.html)):

| 상태 | 의미 / 주의 |
| --- | --- |
| `Success` | 종료 코드 0. 스크립트는 기본적으로 **마지막 명령의 종료 코드**를 반환하므로 중간 실패가 묻힐 수 있다 |
| `Failed` | 0이 아닌 종료 코드 |
| `DeliveryTimedOut` | 정해진 시간 안에 노드에 전달되지 못함 |
| `ExecutionTimedOut` | 실행 시간 초과(실패로 집계) |
| `Undeliverable`, `InvalidPlatform`, `AccessDenied` | **max-errors에 집계되지 않음** |
| `RateExceeded` | 계정의 대기 중인 호출(pending invocations) 한도 초과 |
| `No Instances In Tag` | 태그에 일치하는 노드가 없음 |

타임아웃:

- CLI의 `--timeout-seconds`(30–2,592,000)는 **전달** 제한 시간이다. 이 시간 안에 실행이 시작되지 않으면 명령은 실행되지 않는다.
- 실행 시간은 문서의 `executionTimeout`이 제한한다(`AWS-RunShellScript` 기본 3600초).
- 같은 AWS 문서 안에서 콘솔 기본 타임아웃을 600초와 3600초로 다르게 적고 있어, 정확한 기본 합계는 확인이 필요하다. ([Monitor commands](https://docs.aws.amazon.com/systems-manager/latest/userguide/monitor-commands.html))

### 사용법 / 예시

```bash
# 태그가 Env=prod인 노드에 10%씩, 오류 1건까지 허용하며 실행
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets "Key=tag:Env,Values=prod" \
  --parameters 'commands=["set -e","sudo dnf -y update openssl","openssl version"]' \
  --max-concurrency 10% \
  --max-errors 1 \
  --timeout-seconds 600 \
  --cloud-watch-output-config CloudWatchOutputEnabled=true,CloudWatchLogGroupName=/ssm/run-command \
  --comment "openssl update" \
  --query Command.CommandId --output text

# 노드별 결과 요약
aws ssm list-command-invocations --command-id <command-id> --details \
  --query 'CommandInvocations[].{id:InstanceId,status:Status}'

# 특정 노드의 출력(앞 24,000자)
aws ssm get-command-invocation --command-id <command-id> --instance-id i-0123456789abcdef0
```

- `set -e`를 먼저 넣으면 중간 명령이 실패할 때 바로 0이 아닌 코드로 끝나므로 `Success`가 중간 실패를 숨기지 않는다.
- `get-command-invocation --plugin-name`에는 `aws:RunShellScript` 같은 plugin 이름이 아니라 **step 이름**을 넣어야 한다.

### 시크릿을 명령에 넘기는 방법

Command document는 SecureString 참조를 지원하지 않는다. AWS 예시처럼 평문을 파라미터로 넘기지 말고 **노드 안에서 Parameter Store를 조회**한다. send-command 응답에도 `Parameters`가 그대로 되돌아온다. ([Using parameters in Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-param-runcommand.html))

```bash
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets "Key=tag:Role,Values=app" \
  --parameters 'commands=["export DB_PASSWORD=$(aws ssm get-parameter --name /myapp/prod/db/password --with-decryption --query Parameter.Value --output text)","./migrate.sh"]'
```

이 경우 노드의 instance profile에 해당 파라미터의 `ssm:GetParameter`와 `kms:Decrypt` 권한이 있어야 한다.

## Automation

### 런북 구조

Automation runbook은 `schemaVersion: '0.3'`의 Automation 타입 문서다. `mainSteps`를 순서대로 실행하며, 각 step은 action 하나를 쓴다. ([Automation actions](https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-actions.html))

| 자주 쓰는 action | 용도 |
| --- | --- |
| `aws:executeAwsApi` | 임의의 AWS API 호출 |
| `aws:waitForAwsResourceProperty` | 리소스 상태가 원하는 값이 될 때까지 대기 |
| `aws:assertAwsResourceProperty` | 리소스 상태 검증 |
| `aws:runCommand` | 노드에 Run Command 실행 |
| `aws:executeScript` | Python/PowerShell 스크립트 실행(최대 10분, 출력 100 KB) |
| `aws:branch` | 조건 분기 |
| `aws:approve` | 사람 승인 대기 |
| `aws:sleep`, `aws:pause` | 대기·일시 정지 |
| `aws:invokeLambdaFunction` | Lambda 호출(최대 5분, 출력 200 KB) |
| `aws:executeAutomation` | 다른 runbook 중첩 실행(최대 5단계) |

공통 속성: `maxAttempts`(기본 1), `timeoutSeconds`, `onFailure`(`Abort` 기본 / `Continue` / `step:<name>`), `isEnd`, `nextStep`, `isCritical`.

### 권한: assume role

- 기본적으로 Automation은 **실행한 사용자의 권한**으로 돈다. AWS는 `AmazonSSMAutomationRole` 정책을 붙인 service role(assume role)을 쓰도록 권장한다. 이렇게 하면 사용자에게 EC2 권한 없이 런북 실행 권한만 줄 수 있다("delegated administration"). ([Run an automation](https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-working-executing.html))
- 12시간 넘게 걸리는 automation은 사용자 세션이 만료되므로 반드시 assume role을 써야 한다. (같은 출처)
- 런북이 호출하는 서비스 API(`aws:executeAwsApi` 등)는 모두 role 정책에서 허용해야 한다.

### 실행 모드

| 모드 | 설명 |
| --- | --- |
| Simple | 대상 1개에 한 번 실행 |
| Step-by-step | 단계별로 수동 진행 |
| Rate control | `--targets` + `--max-concurrency` / `--max-errors`로 여러 리소스에 분산 실행 |
| Multi-account / Multi-Region | 중앙 계정의 `AWS-SystemsManager-AutomationAdministrationRole`이 대상 계정의 `AWS-SystemsManager-AutomationExecutionRole`을 assume. `--target-locations`로 지정 ([문서](https://docs.aws.amazon.com/systems-manager/latest/userguide/running-automations-multiple-accounts-regions.html)) |

### 사용법 / 예시

AWS 제공 런북 실행:

```bash
aws ssm start-automation-execution \
  --document-name AWS-RestartEC2Instance \
  --parameters "InstanceId=i-0123456789abcdef0"

aws ssm get-automation-execution --automation-execution-id <execution-id> \
  --query 'AutomationExecution.{status:AutomationExecutionStatus,steps:StepExecutions[].{n:StepName,s:StepStatus}}'
```

직접 만든 최소 런북(인스턴스 중지 후 `stopped`가 될 때까지 대기). [Automation actions](https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-actions.html)의 공식 예시를 바탕으로 했다.

```yaml
# stop-and-wait.yml
schemaVersion: '0.3'
description: Stop an EC2 instance and wait until it is stopped.
assumeRole: '{{ AutomationAssumeRole }}'
parameters:
  AutomationAssumeRole:
    type: String
    default: ''
  InstanceId:
    type: String
mainSteps:
  - name: stopInstance
    action: aws:executeAwsApi
    inputs:
      Service: ec2
      Api: StopInstances
      InstanceIds:
        - '{{ InstanceId }}'
  - name: waitStopped
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 300
    inputs:
      Service: ec2
      Api: DescribeInstances
      InstanceIds:
        - '{{ InstanceId }}'
      PropertySelector: '$.Reservations[0].Instances[0].State.Name'
      DesiredValues:
        - stopped
    isEnd: true
```

```bash
aws ssm create-document \
  --name StopAndWait \
  --document-type Automation \
  --document-format YAML \
  --content file://stop-and-wait.yml

aws ssm start-automation-execution \
  --document-name StopAndWait \
  --parameters "InstanceId=i-0123456789abcdef0,AutomationAssumeRole=arn:aws:iam::111122223333:role/MyAutomationRole"
```

- `assumeRole`을 빈 문자열로 두면 실행한 사용자의 권한으로 동작한다.
- 이 런북은 실제 계정에서 실행해 보지 않았다(확인 필요). 필드 이름은 공식 문서 예시와 대조했다.

## 이슈 / 주의

- **Automation free tier 없음**: 2025-12-31에 기존 사용자의 free tier까지 종료됐다. `aws:waitForAwsResourceProperty`나 루프 step을 많이 쓰는 런북은 step 수만큼 과금되므로, 플릿 전체에 rate control로 돌리면 비용이 빠르게 늘어난다.
- **Automation 한도**: 동시 실행 100개(adaptive concurrency로 최대 500), 대기열 5,000, rate control 실행은 동시 25개. `StartAutomationExecution`은 1 TPS, `GetAutomationExecution`은 3 TPS. 실행 기록은 30일 보관. ([Quotas](https://docs.aws.amazon.com/general/latest/gr/ssm.html))
- **Run Command 전용 한도는 quotas 페이지에 별도 섹션이 없다**(확인 필요). 대기 호출이 많으면 `RateExceeded` 상태가 날 수 있다.
- **Run Command API는 eventually consistent**하다. 명령을 보낸 직후 조회하면 아직 반영되지 않았을 수 있다. ([Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command.html))
- **Command injection**: 사용자 입력을 `{{ param }}`으로 셸에 그대로 치환하는 사용자 정의 Command document는 위험하다. `allowedPattern`/`allowedValues`로 제한하거나 schema 2.2의 `interpolationType: ENV_VAR`를 쓴다. (→ [02-architecture.md](./02-architecture.md#ssm-document))

## 출처

- https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/send-commands-multiple.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/monitor-commands.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/running-commands-console.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-param-runcommand.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/troubleshooting-remote-commands.html
- https://docs.aws.amazon.com/cli/latest/reference/ssm/send-command.html
- https://docs.aws.amazon.com/cli/latest/reference/ssm/get-command-invocation.html
- https://docs.aws.amazon.com/systems-manager/latest/APIReference/API_CommandPlugin.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-actions.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-working-executing.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/running-automations-multiple-accounts-regions.html
- https://docs.aws.amazon.com/cli/latest/reference/ssm/create-document.html
- https://docs.aws.amazon.com/general/latest/gr/ssm.html
- https://aws.amazon.com/systems-manager/pricing/

---
*조사 기준일: 2026-10-12. CLI 예시는 공식 CLI 레퍼런스와 대조했지만 실제 계정에서 실행하지는 않았다.*
