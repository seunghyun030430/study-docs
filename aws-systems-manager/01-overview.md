# AWS Systems Manager 개요

> AWS Systems Manager(SSM)는 EC2·온프레미스·멀티클라우드 서버("managed node")를 **SSM Agent + IAM + 아웃바운드 HTTPS**만으로 한곳에서 접속·명령 실행·설정 관리·자동화하는 운영 서비스 묶음이다. 실무에서 가장 많이 쓰는 기능은 Session Manager(접속), Parameter Store(설정·시크릿), Run Command / Automation(원격 실행·런북)이다.

## 개요 / 정의

> "helps you centrally view, manage, and operate nodes at scale in AWS, on-premises, and multicloud environments." — [What is AWS Systems Manager?](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html)

- 예전 이름은 "Amazon Simple Systems Manager(SSM)"와 "Amazon EC2 Systems Manager"였다. 그래서 지금도 엔드포인트(`ssm.<region>.amazonaws.com`), CLI(`aws ssm`), CloudFormation 타입(`AWS::SSM::*`)에 `SSM`이 남아 있다. (같은 출처)
- 단일 서비스라기보다 **여러 도구(tool)의 모음**이다. 대부분의 도구가 SSM Agent, managed node, SSM Document라는 공통 기반을 공유한다(→ [02-architecture.md](./02-architecture.md)).

## 기능 지도

AWS 사용자 가이드는 도구를 4개 범주로 묶는다. ([Using Systems Manager tools](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-tools.html))

| 범주 | 도구 | 한 줄 설명 |
| --- | --- | --- |
| Node tools | **Session Manager** | 포트를 열지 않고 IAM 기반으로 셸 접속·포트포워딩 |
| | **Run Command** | 여러 노드에 명령·스크립트를 원격으로 일괄 실행 |
| | State Manager | 노드를 원하는 상태로 주기적으로 유지(association) |
| | Patch Manager | OS·애플리케이션 패치 스캔·설치 |
| | Fleet Manager | 노드 목록·파일시스템·로그 등 원격 관리 UI |
| | Inventory / Compliance | 소프트웨어 인벤토리 수집, 패치·설정 준수 현황 |
| | Hybrid Activations | 온프레미스·타 클라우드 서버를 managed node로 등록 |
| | Distributor | 소프트웨어 패키지 배포 |
| Change Management tools | **Automation** | 런북(runbook)으로 AWS 리소스·노드 작업 자동화 |
| | Documents | 모든 도구가 쓰는 작업 정의(SSM Document) |
| | Maintenance Windows | 정해진 시간대에 작업 예약 실행 |
| | Change Calendar | 변경 허용/금지 기간 정의 |
| | Quick Setup | 권장 구성을 계정·조직에 일괄 배포(Host Management, Patch Policy, **Resource Scheduler** 등 11개 유형) → [06](./06-quick-setup.md) |
| | Change Manager | 변경 요청·승인 흐름(**2025-11-07부터 신규 고객 가입 중단**) |
| Application tools | **Parameter Store** | 계층형 설정값·시크릿 저장 |
| | AppConfig | 애플리케이션 설정·feature flag 배포 |
| | Application Manager | 애플리케이션 단위 리소스 뷰(**신규 고객 가입 중단**) |
| Operations tools | Explorer / OpsCenter | 운영 데이터 집계, 운영 이슈(OpsItem) 관리 |
| | Incident Manager | 인시던트 대응(**2025-11-07부터 신규 고객 가입 중단**) |
| | CloudWatch Dashboards | **2026-04-30 이후 제공 종료**, CloudWatch 콘솔 사용 |

굵게 표시한 4개(Session Manager, Run Command, Automation, Parameter Store)를 이 레포에서 깊게 다룬다.

### 최근 변화(2024–2026)

| 날짜 | 변화 | 출처 |
| --- | --- | --- |
| 2024-11-21 | 통합 콘솔(unified console) 출시: 여러 계정·리전의 노드 통합 조회, node insights, Amazon Q 자연어 질의, 미관리 노드 진단·복구 | [Unified console](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-unified-console.html), [Release history](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-release-history.html) |
| 2025-04-29 | Just-in-time node access(기한부 노드 접근 승인, Windows RDP 세션 녹화) | [Release history](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-release-history.html) |
| 2025-11-07 | Change Manager, Incident Manager 신규 고객 가입 중단 | [Change Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/change-manager-availability-change.html), [Incident Manager](https://docs.aws.amazon.com/incident-manager/latest/userguide/incident-manager-availability-change.html) |
| 2026-04-30 | Systems Manager CloudWatch Dashboards 제공 종료 | 사용자 가이드 공지 배너 |
| 2026-06-30 | 하이브리드 노드 **advanced-instances tier 폐지**(1,000대 제한 해제), 대신 사용량 기반 과금으로 전환 | [Hybrid activations](https://docs.aws.amazon.com/systems-manager/latest/userguide/activations.html), [Pricing](https://aws.amazon.com/systems-manager/pricing/) |
| 2026-07 | Microsoft Azure VM 지원(Cloud Connector로 SSM Agent 배포) | [What's New](https://aws.amazon.com/about-aws/whats-new/2026/06/aws-systems-manager-multicloud-vm/) |
| 2026-12-31 | 기존 Session Manager / Hybrid Activations 콘솔 페이지 지원 종료 예정(통합 콘솔로 이전) | [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html), [Activations](https://docs.aws.amazon.com/systems-manager/latest/userguide/activations.html) |

- advanced-instances tier 폐지일은 요금·사용자 가이드에는 2026-06-30, release history에는 2026-07-01로 적혀 있다.
- Application Manager의 신규 가입 중단 시점(2026-07-30)은 release history에서만 확인했다(확인 필요).

## Managed node

> "A managed node is any machine configured for use with Systems Manager in hybrid and multicloud environments." — [Managed nodes](https://docs.aws.amazon.com/systems-manager/latest/userguide/fleet-manager-managed-nodes.html)

- 대상: EC2 인스턴스, 온프레미스 서버, VM(다른 클라우드 포함), AWS IoT Greengrass core 디바이스, 엣지 디바이스.
- ID 형식: EC2는 `i-...`, hybrid activation으로 등록한 머신은 `mi-...`.
- 용어: 예전에는 "managed instance"라고 불렀다. 지금 "instance"는 EC2만 가리킨다.

managed node가 되려면 세 가지 조건이 필요하다. 자세한 내용은 [02-architecture.md](./02-architecture.md)에서 다룬다.

| 조건 | 내용 |
| --- | --- |
| SSM Agent | 노드에 설치·실행 중이어야 함(주요 AWS AMI에는 기본 설치) |
| IAM 권한 | EC2: instance profile(`AmazonSSMManagedInstanceCore`) 또는 DHMC. 비 EC2: service role |
| 네트워크 | `ssm`, `ssmmessages`(+ 구형 리전은 `ec2messages`) 엔드포인트로 아웃바운드 443. 인바운드 불필요 |

## 요금 개요

출처: [Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/) (조사 기준일 시점 값)

| 기능 | 요금 |
| --- | --- |
| Session Manager / Run Command / Patch Manager (EC2) | 추가 요금 없음 |
| Session Manager (하이브리드·멀티클라우드 노드) | 2026-09-30부터 세션당 $0.05 |
| Run Command (하이브리드·멀티클라우드 노드) | 2026-09-30부터 호출(invocation)당 $0.002 |
| State Manager, Inventory, Fleet Manager, Compliance, Maintenance Windows | 추가 요금 없음 |
| Parameter Store Standard | 무료(높은 처리량을 켜면 API 10,000건당 $0.05) |
| Parameter Store Advanced | 파라미터당 월 $0.05 + API 10,000건당 $0.05 |
| Automation | step당 $0.002, `aws:executeScript`는 초당 $0.00003 추가. **현재 free tier 없음** |
| Just-in-time node access | 노드-시간당 $0.0137부터(구간별 할인) |
| OpsCenter | OpsItem 1,000개당 $2.97 등 |

- Automation free tier는 2025-08-14부터 신규 사용자에게 제공되지 않았고, 기존 사용자도 2025-12-31에 종료됐다. ([Automation](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html))
- 요금 외에 KMS, S3, CloudWatch Logs 등 연계 서비스 비용이 따로 발생한다.

## 어떤 기능을 언제 쓰나

| 하고 싶은 일 | 기능 | 문서 |
| --- | --- | --- |
| private 서브넷 서버에 SSH 없이 접속, RDS로 터널링 | Session Manager | [03-session-manager.md](./03-session-manager.md) |
| 설정값·DB 비밀번호 저장, 최신 AMI ID 조회 | Parameter Store | [04-parameter-store.md](./04-parameter-store.md) |
| 태그로 묶인 서버 100대에 스크립트 일괄 실행 | Run Command | [05-run-command-automation.md](./05-run-command-automation.md) |
| "인스턴스 중지 → 스냅샷 → 재시작" 같은 다단계 작업 자동화 | Automation | [05-run-command-automation.md](./05-run-command-automation.md) |
| 새 계정·리전의 EC2를 한 번에 managed node로 만들고 기본 관리 자동화 | Quick Setup Host Management | [06-quick-setup.md](./06-quick-setup.md#host-management) |
| 개발 서버를 업무 시간에만 켜기(EC2 자동 켜기·끄기) | Quick Setup Resource Scheduler | [06-quick-setup.md](./06-quick-setup.md#resource-scheduler) |
| 노드를 주기적으로 원하는 상태로 유지 | State Manager | 이 레포 범위 밖 |
| 정기 패치 | Patch Manager + Maintenance Windows | 이 레포 범위 밖 |

## 출처

- https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-tools.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-unified-console.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-release-history.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/fleet-manager-managed-nodes.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/activations.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/change-manager-availability-change.html
- https://docs.aws.amazon.com/incident-manager/latest/userguide/incident-manager-availability-change.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/application-manager.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html
- https://aws.amazon.com/systems-manager/pricing/
- https://aws.amazon.com/about-aws/whats-new/2026/06/aws-systems-manager-multicloud-vm/

---
*조사 기준일: 2026-10-12. 요금과 기능 제공 여부는 자주 바뀌므로 적용 전에 요금 페이지와 release history를 다시 확인할 것.*
