# 비용·IP·운영 이슈

> 기존 요금은 **1,000통당 $0.10 + 첨부 GB당 $0.12**다. 2026-07-21부터 **Essentials·Pro·Enterprise 요금제**가 생겼다. 그 뒤 새로 시작하는 계정·리전은 Essentials($0.16/1,000통)로 시작하고, 기존 사용자는 고르기 전까지 기존 요금이다. 발송 IP는 **공유 IP가 기본이자 대부분의 정답**이다. 꾸준하고 많은 발송이 있을 때만 전용 IP를 쓰고, 그중에서도 **managed 전용 IP**가 관리 부담이 적다. 여러 고객사를 대신 발송한다면 **테넌트**로 평판을 나눈다. 운영 사고는 대부분 인증 누락, 바운스 미처리, 샌드박스·리전 착오, throttling 재시도 누락이다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[04](./04-reputation-limits.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 공유 IP | 여러 SES 고객이 함께 쓰는 발송 IP. AWS가 평판을 관리 |
| 전용 IP(standard) | 내 계정만 쓰는 고정 IP. 직접 워밍업·평판 관리 |
| 전용 IP(managed) | 내 계정 전용이지만 AWS가 ISP별 워밍업·개수 조정을 자동으로 함 |
| IP 워밍업 | 새 IP의 발송량을 몇 주에 걸쳐 늘려 메일함 사업자의 신뢰를 쌓는 과정 |
| 전용 IP 풀 | 전용 IP 묶음. configuration set으로 어떤 메일이 어느 풀을 쓸지 정함 |
| 테넌트(tenant) | 한 계정 안에서 identity·configuration set·템플릿·평판을 분리하는 단위 |
| Mail Manager | 수신 메일 게이트웨이(필터링, 릴레이, 보관) 기능 |

## 요금

출처: [SES pricing](https://aws.amazon.com/ses/pricing/), [Pricing plans](https://docs.aws.amazon.com/ses/latest/dg/pricing-plans.html), [What's New 2026-07-21](https://aws.amazon.com/about-aws/whats-new/2026/07/amazon-ses-pricing-plans/)

### 기존 요금(à la carte)

| 항목 | 금액 |
| --- | --- |
| 발송 | $0.10 / 1,000통 |
| 첨부 | $0.12 / GB |
| 수신 | $0.10 / 1,000통 + $0.09 / 1,000 chunk(256 KB) |
| 전용 IP(standard) | IP당 월 $24.95 |
| 전용 IP(managed) | 계정당 월 $15 + 1,000통당 $0.08(월 1,000만 통까지) |
| VDM SES deliverability | 1,000통당 $0.07(월 1,000만 통까지) |
| 테넌트 | 테넌트당 월 $0.005 + 1,000통당 $0.005 |

- 발송에 **실패한 호출은 과금되지 않는다.**
- SES 전용 무료 사용량(예전의 월 3,000통)은 요금 페이지에서 사라졌다. 대신 신규 AWS 고객에게 최대 $200의 Free Tier 크레딧(12개월)이 있고, SES에도 쓸 수 있다.
- 요금 페이지에는 리전별 차이가 표시돼 있지 않다. 서울이 같은 금액인지는 Price List API로 확인하지 않았다(확인 필요).

### 요금제(2026-07-21+)

| 요금제 | 고정비(계정·리전당) | 1,000통당(월 0–1,000만 / 1,000만–1억 / 1억 초과) | 포함되는 것(요약) |
| --- | --- | --- | --- |
| Essentials | 없음 | $0.16 / $0.14 / $0.11 | 도달률 인사이트 |
| Pro | 월 $105 | $0.22 / $0.17 / $0.12 | managed 전용 IP, 이메일 검증, 받은편지함 배치 가시성 |
| Enterprise | 월 $500 | $0.23 / $0.18 / $0.13 | 멀티 리전 복원력, 워크로드 단위 평판 분리, 연간 도달률 평가 |

- 요금제는 **계정·리전 단위**다. 요금제를 골라도 기능이 자동으로 켜지지 않는다.
- 2026-07-21 이후 새로 시작한 계정(또는 2025-06-01 이후 SES 사용이 없던 계정·리전)은 Essentials로 시작한다는 요금 페이지 주석이 있다. 나머지는 고르기 전까지 기존 요금이다.
- Pro의 Global deliverability 포함 범위는 요금 페이지의 표와 주석이 서로 달라 확인 필요다.

### 어느 쪽이 싼가

월 100만 통, 첨부 없음 기준(계산 예):

| 방식 | 계산 | 월 비용 |
| --- | --- | --- |
| 기존 | 1,000 × $0.10 | **$100** |
| 기존 + VDM | $100 + 1,000 × $0.07 | $170 |
| Essentials | 1,000 × $0.16 | $160 |
| Pro | $105 + 1,000 × $0.22 | $325 |

- 순수 발송 단가만 보면 기존 요금이 가장 싸다. 요금제는 VDM·managed IP·이메일 검증 같은 기능을 함께 쓸 때 비교한다.
- 위 계산은 가정에서 계산한 예시다. 실제 포함 기능과 금액은 요금 페이지와 AWS Pricing Calculator로 확인한다.

## 공유 IP vs 전용 IP

출처: [Dedicated IPs](https://docs.aws.amazon.com/ses/latest/dg/dedicated-ip.html), [IP warming](https://docs.aws.amazon.com/ses/latest/dg/dedicated-ip-warming.html)

| 항목 | 공유 IP(기본) | 전용 standard | 전용 managed |
| --- | --- | --- | --- |
| 평판 | AWS가 여러 고객 것을 함께 관리 | **내 발송만으로** 결정 | 내 발송만으로 |
| 워밍업 | 불필요 | 필요(자동 워밍업 기본 켜짐, 45일에 걸쳐 증가) | ISP별 자동 |
| IP 고정 | 아님 | **고정**(알려진 IP) | 바뀔 수 있음 |
| 신청 | — | 지원 케이스 | 설정에서 켜기 |
| 맞는 경우 | **대부분의 트랜잭셔널** | 많은 양을 꾸준히, IP를 상대방에 등록해야 함 | 많지만 불규칙한 양 |

> "If you don't plan to send large volumes of email on a regular and predictable basis, we recommend that you use shared IP addresses." — SES Dedicated IPs

- 전용 IP는 **발송량이 적으면 오히려 불리하다.** 메일함 사업자가 판단할 데이터가 부족하다. AWS 문서는 워밍업 후 IP마다 메일함 사업자별로 하루 1,000통 정도를 권한다.
- 수동 워밍업은 사업자에 따라 **2주~6주**가 걸린다.
- 마케팅과 트랜잭셔널을 **다른 전용 IP 풀**로 나누면 평판을 분리할 수 있다. 공유 IP를 쓴다면 서브도메인과 configuration set으로 나눈다.

## 리전

출처: [Regions](https://docs.aws.amazon.com/ses/latest/dg/regions.html), [SES endpoints](https://docs.aws.amazon.com/general/latest/gr/ses.html)

서울(ap-northeast-2)은 API(`email.ap-northeast-2.amazonaws.com`), SMTP(`email-smtp.ap-northeast-2.amazonaws.com`), 수신(`inbound-smtp.ap-northeast-2.amazonaws.com`)을 모두 지원한다.

**리전마다 따로인 것**: 샌드박스 상태, 발송 한도, identity와 DKIM, suppression list, 피드백 설정, SMTP 자격증명, 요금제, 테넌트. SNS 토픽도 같은 리전이어야 한다.

- 리전 장애에 대비해 두 리전에서 보내려면 **모든 것을 두 번** 설정해야 한다. Enterprise 요금제의 멀티 리전 기능은 이를 돕는 것으로 소개되지만 동작은 이번에 자세히 확인하지 않았다.

## 테넌트

출처: [Tenants](https://docs.aws.amazon.com/ses/latest/dg/tenants.html)

SaaS처럼 **고객사마다 메일을 대신 보내는 서비스**는 한 고객사의 나쁜 발송이 계정 전체를 정지시킬 수 있다. 테넌트는 identity·configuration set·템플릿·평판을 고객사별로 나눈다.

| 항목 | 내용 |
| --- | --- |
| 평판 정책 | Standard(기본, 심각한 문제에 테넌트 정지) / Strict(모든 문제에 정지) / None(기록만) |
| 발송 | API `--tenant-name`, SMTP는 `X-SES-TENANT` 헤더 |
| 이벤트 | 테넌트 정지·재개는 **EventBridge 이벤트가 있음**(계정 정지는 없음) |
| 한도 | 기본 10,000개, 리전 단위 |

> "their combined sending activity still affects your overall account reputation." — SES Tenants

테넌트는 고객사 하나를 먼저 멈춰 피해를 줄이는 장치다. 계정 평판에서 완전히 떼어 내 주지는 않는다.

## 모니터링·알람

| 감시 대상 | 방법 | 문서 |
| --- | --- | --- |
| 바운스율·불만율 | `AWS/SES` `Reputation.BounceRate`·`Reputation.ComplaintRate` 알람 | [04](./04-reputation-limits.md#감시) |
| 계정 정지 | `GetAccount` `EnforcementStatus` 주기 확인, AWS Health | [04](./04-reputation-limits.md#감시) |
| 한도 사용률 | `SentLast24Hours` ÷ `Max24HourSend` | [04](./04-reputation-limits.md#발송-한도) |
| 개별 바운스·불만 | configuration set → SNS/EventBridge → 처리 | [03](./03-sending.md#configuration-set과-이벤트) |
| Gmail 스팸 신고율 | Google Postmaster Tools | [02](./02-identity-authentication.md#gmailyahoo-발송자-요구사항) |
| 인증 설정 문제 | VDM Advisor | [04](./04-reputation-limits.md#virtual-deliverability-managervdm) |
| 렌더링 실패 | `RenderingFailure` 이벤트 | [03](./03-sending.md#템플릿) |

알람 기본은 [aws-cloudwatch/03](../aws-cloudwatch/03-alarms.md)을 참고한다.

## 흔한 사고

출처: [Troubleshooting error messages](https://docs.aws.amazon.com/ses/latest/dg/troubleshoot-error-messages.html), [Sending quota errors](https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas-errors.html)

| 증상 | 원인 | 대응 |
| --- | --- | --- |
| `MessageRejected: Email address is not verified` | 보내는 주소가 그 리전에서 미검증, 또는 **샌드박스에서 받는 주소 미검증** | 리전 확인, identity 검증, 프로덕션 액세스 요청 |
| `Account is paused` | 바운스·불만율 초과 | 지원 케이스에 원인·조치·재발 방지 답변 |
| `ThrottlingException` / `454 Throttling failure` | 초당 속도·24시간 한도 초과 | 앱 속도 제한, 재시도, 한도 증가 요청 |
| `Maximum SigV2 SMTP sending rate exceeded` | 2019-01-10 이전에 만든 SMTP 자격증명 | 새 SMTP 자격증명 발급 |
| `MailFromDomainNotVerified` | custom MAIL FROM의 MX를 못 찾음 + `REJECT_MESSAGE` | MX 레코드 확인 |
| 메일이 스팸함으로 | DKIM·DMARC 없음, 정렬 안 됨, 평판 저하, 내용 | [02](./02-identity-authentication.md) 설정, Postmaster Tools 확인 |
| 같은 메일이 두 번 | 타임아웃 후 재시도 | 앱에서 멱등 처리 |
| 수신자가 메일을 못 받음, 오류는 없음 | 주소가 suppression list에 있음(받아들이지만 안 보냄) | 이벤트의 `OnAccountSuppressionList` 확인 |

## 수신 기능 개요

출처: [Receipt rule actions](https://docs.aws.amazon.com/ses/latest/dg/receiving-email-action.html), [Mail Manager](https://docs.aws.amazon.com/ses/latest/dg/eb.html)

| 기능 | 하는 일 | 메모 |
| --- | --- | --- |
| 수신 규칙(receipt rule) | 도메인으로 오는 메일을 받아 S3 저장, Lambda 호출, SNS 알림, 바운스, WorkMail 전달 | 서울 포함 22개 리전. SNS 액션은 150 KB 초과 메일을 바운스시킴 → 큰 메일은 S3 |
| Mail Manager | 메일 게이트웨이: ingress endpoint, 트래픽 정책, 규칙, SMTP 릴레이, 보관, 보안 애드온 | ingress endpoint당 월 $50 + 처리 1,000통당 $0.15 |

- 고객 문의 메일을 받아 티켓으로 만들기, 바운스 메일 회신 처리 같은 용도로 쓴다.
- 수신하려면 도메인 MX를 `inbound-smtp.<region>.amazonaws.com`으로 지정한다. 기존 메일 서비스(Google Workspace 등)를 쓰는 도메인이라면 **수신용 서브도메인**을 따로 둔다(그렇지 않으면 기존 메일이 끊긴다).

## 안티패턴

| 안티패턴 | 결과 | 대안 | 근거 |
| --- | --- | --- | --- |
| 이메일 주소 identity로 운영 | DKIM 정렬 없음 → 스팸함 | 도메인 identity + Easy DKIM | [02](./02-identity-authentication.md) |
| DMARC 없이 대량 발송 | Gmail·Yahoo 거부 | `p=none`부터 단계적으로 | [02](./02-identity-authentication.md#dmarc) |
| 바운스·불만 이벤트 미처리 | 바운스율 상승 → 계정 정지 | configuration set + SNS + DB 갱신 | [04](./04-reputation-limits.md#바운스불만-처리-파이프라인) |
| 마케팅과 트랜잭셔널을 같은 도메인·설정으로 | 마케팅 신고가 비밀번호 재설정 메일까지 영향 | 서브도메인·configuration set·IP 풀 분리 | [02](./02-identity-authentication.md#dmarc) |
| 발송량 적은데 전용 IP | 평판 데이터 부족, 월 비용 | 공유 IP | 이 문서 |
| 실제 주소로 부하 테스트 | 바운스 폭증 | 메일박스 시뮬레이터 | [03](./03-sending.md#api-v2-vs-smtp) |
| throttling 무시 | 메일 유실 | 앱 쪽 속도 제한 + 재시도 | [04](./04-reputation-limits.md#한도-초과throttling) |
| `ses:FromAddress` 제한 없는 키 | 유출 시 도메인 피싱 | 조건 키로 주소 제한 | [03](./03-sending.md#iam-권한) |
| 루트 도메인 MX를 SES 수신으로 변경 | 기존 회사 메일 중단 | 수신용 서브도메인 | 이 문서 |
| 계정 정지를 이벤트로만 감시 | 정지를 늦게 앎 | `EnforcementStatus` 확인, AWS Health | [04](./04-reputation-limits.md#감시) |

## 출처

- https://aws.amazon.com/ses/pricing/
- https://docs.aws.amazon.com/ses/latest/dg/pricing-plans.html
- https://aws.amazon.com/about-aws/whats-new/2026/07/amazon-ses-pricing-plans/
- https://docs.aws.amazon.com/ses/latest/dg/dedicated-ip.html
- https://docs.aws.amazon.com/ses/latest/dg/dedicated-ip-warming.html
- https://docs.aws.amazon.com/ses/latest/dg/regions.html
- https://docs.aws.amazon.com/general/latest/gr/ses.html
- https://docs.aws.amazon.com/ses/latest/dg/tenants.html
- https://docs.aws.amazon.com/ses/latest/dg/troubleshoot-error-messages.html
- https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas-errors.html
- https://docs.aws.amazon.com/ses/latest/dg/receiving-email-action.html
- https://docs.aws.amazon.com/ses/latest/dg/receiving-email-action-sns.html
- https://docs.aws.amazon.com/ses/latest/dg/eb.html

---
*조사 기준일: 2026-10-28. 요금은 요금 페이지 표시 금액이고, 계산 예는 가정에서 계산한 것이다.*
