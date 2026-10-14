# 이슈·한계·안티패턴

> 실무 장애의 대부분은 **"노드가 managed node로 안 보임"**(agent·IAM·네트워크 3요소 중 하나 누락), **API 쓰로틀링**(Parameter Store 기본 40 TPS 등), **과도한 권한**(`Resource: "*"`의 `SendCommand`/`StartSession`) 세 가지다. 진단은 `ssm-cli get-diagnostics`와 `AWSSupport-TroubleshootManagedInstance` 런북으로 시작한다.

## 1. 노드가 managed node로 보이지 않음

출처: [Troubleshooting managed node availability](https://docs.aws.amazon.com/systems-manager/latest/userguide/fleet-manager-troubleshooting-managed-nodes.html)

AWS 문서가 제시하는 점검 항목:

| # | 점검 항목 | 확인 방법 / 비고 |
| --- | --- | --- |
| 1 | SSM Agent 설치·실행 | `sudo systemctl status amazon-ssm-agent` (snap 설치 Ubuntu는 `snap.amazon-ssm-agent.amazon-ssm-agent`) |
| 2 | IAM 권한 | instance profile에 `AmazonSSMManagedInstanceCore`, 또는 DHMC 활성화 |
| 3 | 엔드포인트 연결 | `ssm`/`ssmmessages`(+`ec2messages`)로 아웃바운드 443, 또는 VPC endpoint |
| 4 | OS 지원 여부 | 문서·기능별 지원 OS 확인 |
| 5 | 리전 | 콘솔·CLI와 노드가 같은 리전인지 |
| 6 | 프록시 | agent 프록시 설정 |
| 7 | TLS 인증서 | Amazon Root CA 1 / Starfield 루트 인증서 |
| 8 | DHMC 설정 | IMDSv2 필수, agent 3.2.582.0 이상, 적용까지 최대 30분 |
| 9 | Hybrid activation | 만료, 등록 한도 초과, VM 복제로 인한 ID 충돌 |

- 서비스는 5분마다 노드 상태를 확인하고, `Connection Lost` 상태로 30일 넘게 지나면 Fleet Manager 목록에서 빠질 수 있다. (같은 출처)
- DHMC를 켰는데도 안 되는 흔한 원인:
  - 인스턴스가 **IMDSv1만** 허용하고 있다.
  - instance profile이 따로 붙어 있어 그쪽이 우선 쓰이는데, 그 profile에 SSM 권한이 없다.

  ([Instance permissions](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-instance-permissions.html))

### 진단 도구

```bash
# 노드 안에서(agent 3.1.501.0 이상): IMDS, 엔드포인트 연결, 자격증명, 프록시, agent 버전 점검
sudo ssm-cli get-diagnostics --output table
```

([ssm-cli](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-cli.html))

```bash
# 밖에서: 보안 그룹, NACL, 라우팅, VPC endpoint, instance profile 점검 런북
aws ssm start-automation-execution \
  --document-name AWSSupport-TroubleshootManagedInstance \
  --parameters "InstanceId=i-0123456789abcdef0"
```

- 이 런북은 IPv6 규칙은 평가하지 않는다. ([AWSSupport-TroubleshootManagedInstance](https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-awssupport-troubleshoot-managed-instance.html)) 단, 이 내용은 검색 결과 요약으로 확인했고 원문 페이지를 직접 열지는 못했다(확인 필요).
- 통합 콘솔은 미관리 노드를 대상으로 진단·복구 런북을 예약 실행하는 기능을 제공한다. ([Unified console](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-unified-console.html))
- 시계 오차(clock skew)는 서명 오류의 흔한 원인으로 알려져 있지만, 이번에 확인한 공식 문제 해결 페이지에는 별도 항목이 없었다(확인 필요).

## 2. 쓰로틀링과 한도

`ThrottlingException`("Your request rate is too high")은 HTTP 400으로 반환되며, AWS SDK가 자동으로 재시도한다. ([Common errors](https://docs.aws.amazon.com/systems-manager/latest/APIReference/CommonErrors.html))

주요 기본 한도([Quotas](https://docs.aws.amazon.com/general/latest/gr/ssm.html)):

| 영역 | 한도 | 대응 |
| --- | --- | --- |
| Parameter Store Get 계열 | 40 TPS(3개 API 합산) | 앱 캐싱, Higher throughput(유료, `GetParameter` 최대 10,000 TPS) |
| Parameter Store `PutParameter` | 3 TPS(Higher throughput 시 10) | 배포 파이프라인에서 일괄 쓰기를 직렬화·백오프 |
| SecureString | KMS 요청 한도에도 걸림 | KMS 한도 확인 |
| Session Manager `StartSession` | 3 TPS | 대량 자동 접속 스크립트 지양 |
| Automation 동시 실행 | 100(adaptive concurrency로 최대 500), 대기열 5,000 | rate control 사용 |
| `StartAutomationExecution` | 1 TPS | 이벤트 폭주 시 대기열·백오프 |
| SSM Document | 문서당 64 KB, 계정당 500개 | 스크립트는 S3에 두고 문서는 얇게 |
| Managed node | 계정·리전당 약 2,400개가 기본값으로 안내됨 | 확인 필요: advanced tier 폐지(2026-06-30) 이후 적용 방식 |

- 대표적인 쓰로틀링 시나리오는 Lambda·ECS 태스크가 콜드 스타트마다 `GetParameter`를 수십 번 호출하는 경우다. `GetParameters`(최대 10개)나 `GetParametersByPath`로 묶고, 결과를 캐시한다. 단, API 요금의 "interaction" 수는 묶어서 호출해도 줄지 않는다. ([04-parameter-store.md](./04-parameter-store.md#처리량tps))

## 3. 보안 안티패턴

AWS의 [Security best practices](https://docs.aws.amazon.com/systems-manager/latest/userguide/security-best-practices.html)를 기준으로 정리했다. "안티패턴" 열은 모범 사례를 뒤집어 적은 것이다.

| 안티패턴 | 위험 | 권장 |
| --- | --- | --- |
| 사용자에게 `ssm:StartSession` / `ssm:SendCommand`를 `Resource: "*"`로 허용 | 계정 내 모든 노드에서 root 수준 명령 실행 가능 | 문서 ARN과 `ssm:resourceTag/<key>` 조건으로 제한 ([Run Command setup](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command-setting-up.html)) |
| instance role에 `AmazonSSMFullAccess` 등 넓은 정책 부여 | 노드가 탈취되면 SSM 전체 제어 가능 | `AmazonSSMManagedInstanceCore` + 필요한 최소 정책(공식 문서에 이 정책을 금지하는 명시적 문구는 없음, 최소 권한 원칙에서 도출) |
| SecureString에 기본 키 `aws/ssm`만 사용 | `GetParameter` 권한만 있으면 모든 SecureString 복호화 가능 | customer managed KMS 키 + 키 정책 |
| 시크릿을 Run Command 파라미터에 평문으로 전달 | 응답·명령 기록에 노출(CloudTrail 기록 여부는 확인 필요) | 노드에서 Parameter Store 조회 ([05](./05-run-command-automation.md#시크릿을-명령에-넘기는-방법)) |
| 사용자 정의 문서에서 입력값을 셸에 직접 치환 | Command injection | `allowedPattern`, `interpolationType: ENV_VAR` |
| SSM Document를 공개(public) 공유 | 내부 스크립트·구성 노출 | 계정 설정에서 public sharing 차단 |
| Automation을 관리자 계정 권한으로 직접 실행 | 런북이 관리자 권한을 그대로 상속 | 최소 권한 assume role 사용 |
| 포트포워딩·SSH 세션을 감사 대상 접속 경로로 허용 | 세션 로그가 남지 않음 | `AWS-StartSSHSession` 문서 Deny, 셸 세션만 허용 ([03](./03-session-manager.md#이슈--안티패턴--주의)) |
| SSM Agent를 오래 방치 | 보안 패치 누락, 신규 기능 미지원 | AWS 권장: agent·CLI를 최소 2주마다 업데이트(State Manager로 자동화 가능) |

- 글로벌 조건 키로 `ssm:SendCommand`를 제한하는 경우 `aws:ViaAWSService` 조건도 함께 넣어야 한다. ([Run Command setup](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command-setting-up.html))
- 감사는 CloudTrail(API 호출), Session Manager 로그(S3/CloudWatch), AWS Config, EventBridge 알림을 함께 쓴다.

## 4. 기타 운영 주의

- **Run Command `Success`는 성공이 아니다**: 마지막 명령의 종료 코드만 반영된다. `set -e`를 쓰고, 결과를 검증하는 단계를 따로 둔다.
- **태그에 일치하는 노드가 없어도** 명령 상태는 `No Instances In Tag`이거나 경우에 따라 성공처럼 보일 수 있다. 실행 전에 대상 수를 확인한다.
- **출력 잘림**: API 응답 출력은 24,000자(상세 목록은 2,500자)로 잘린다. S3/CloudWatch 출력을 기본으로 켠다.
- **기능 종료·변경 추적**: Change Manager, Incident Manager, Application Manager는 신규 가입이 중단됐고, 하이브리드 노드 과금 체계도 2026년에 바뀌었다. 새로 설계할 때는 [01-overview.md](./01-overview.md)의 "최근 변화" 표를 확인한다.

## 출처

- https://docs.aws.amazon.com/systems-manager/latest/userguide/fleet-manager-troubleshooting-managed-nodes.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-instance-permissions.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-cli.html
- https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-awssupport-troubleshoot-managed-instance.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-unified-console.html
- https://docs.aws.amazon.com/systems-manager/latest/APIReference/CommonErrors.html
- https://docs.aws.amazon.com/general/latest/gr/ssm.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/security-best-practices.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command-setting-up.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/monitor-commands.html

---
*조사 기준일: 2026-10-12. "확인 필요"로 표시한 항목은 공식 문서에서 명시적 근거를 찾지 못한 것이다.*
