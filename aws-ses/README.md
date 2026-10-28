# Amazon SES 정리

Amazon SES(Simple Email Service)를 실무 관점에서 정리한 문서 모음. **트랜잭셔널 발송**(가입 인증, 비밀번호 재설정, 알림) 중심이고, 마케팅 대량 발송과 수신은 개요 수준이다. 중점은 **identity·메일 인증 / 발송 방식·이벤트 / 샌드박스·한도·평판·바운스 처리 / 비용·IP·운영 이슈**. 조사 기준일 **2026-10-28**.

관련 주제: 알람은 [aws-cloudwatch](../aws-cloudwatch/README.md), IAM 정책·조건 키는 [aws-iam](../aws-iam/README.md), Terraform은 [terraform](../terraform/README.md).

## 한눈에 보기

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| 처음에 무엇을 하나? | 도메인 identity 검증(Easy DKIM CNAME 3개) → custom MAIL FROM → DMARC → 샌드박스 해제 요청 | [01](./01-overview.md), [02](./02-identity-authentication.md) |
| Gmail·Yahoo에 잘 가려면? | SPF·DKIM·DMARC + From 정렬, 스팸 신고율 0.3% 미만(권장 0.1%). Gmail은 2025-11부터 집행 강화 | [02](./02-identity-authentication.md) |
| API vs SMTP? | 앱은 **API v2 `SendEmail`**(IAM role). SMTP는 기존 라이브러리·도구용(리전별 SMTP 자격증명) | [03](./03-sending.md) |
| 바운스는 어떻게 아나? | **configuration set → SNS/EventBridge** 이벤트. Terraform `enabled` 기본 false 주의 | [03](./03-sending.md) |
| 샌드박스는? | 리전마다, 검증된 주소에만, 하루 200통·초당 1통. 해제 요청은 96시간 안에 첫 답변 | [04](./04-reputation-limits.md) |
| 계정이 정지되는 기준은? | 바운스 5%(검토)·10%(정지 가능), 불만 0.1%(검토)·0.5%(정지 가능) | [04](./04-reputation-limits.md) |
| 정지를 어떻게 아나? | **EventBridge 이벤트 없음.** `GetAccount` `EnforcementStatus`, AWS Health, 평판 메트릭 알람 | [04](./04-reputation-limits.md) |
| 비용은? | 기존 1,000통당 $0.10. 2026-07-21부터 요금제(Essentials $0.16, Pro 월 $105 + $0.22 등) | [05](./05-cost-operations-issues.md) |
| 전용 IP가 필요한가? | 대부분 **아니다.** 많은 양을 꾸준히 보낼 때만, 그중 managed가 관리 부담 적음 | [05](./05-cost-operations-issues.md) |

| 지표 | 권장 | 검토 | 정지 가능 |
| --- | --- | --- | --- |
| 바운스율 | 2% 미만 | 5% 이상 | 10% 이상 |
| 불만율 | 0.1% 미만 | 0.1% 이상 | 0.5% 이상 |

| DNS 레코드 | 이름 | 값 |
| --- | --- | --- |
| DKIM(×3) | `<token>._domainkey.example.com` | CNAME `<token>.dkim.amazonses.com`(일부 리전은 `dkim.<region>.amazonses.com`) |
| MAIL FROM MX | `mail.example.com` | `10 feedback-smtp.<region>.amazonses.com` |
| MAIL FROM SPF | `mail.example.com` | TXT `v=spf1 include:amazonses.com ~all` |
| DMARC | `_dmarc.example.com` | TXT `v=DMARC1; p=none; rua=mailto:...` |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요(단계별 입문) — 메일 전달 구조, 트랜잭셔널·마케팅, identity, SPF·DKIM·DMARC, 샌드박스, 바운스·불만, 발송 방법, 기능 지도, 요금 개요, 용어 정리 |
| [02-identity-authentication.md](./02-identity-authentication.md) | 도메인·주소 identity, Easy DKIM·BYODKIM, custom MAIL FROM·SPF, DMARC 단계적 적용, Gmail·Yahoo 요구사항, Terraform + Route 53 예시 |
| [03-sending.md](./03-sending.md) | API v2 vs SMTP, 크기·수신자 한도, 템플릿(Handlebars), configuration set·이벤트 목적지·추적, 구독 해지 헤더, IAM 조건 키·sending authorization |
| [04-reputation-limits.md](./04-reputation-limits.md) | 샌드박스 해제, 발송 한도·throttling, 평판 임계치·계정 정지·감시, suppression list, 바운스·불만 처리 파이프라인, VDM |
| [05-cost-operations-issues.md](./05-cost-operations-issues.md) | 요금(기존·요금제), 공유·전용 IP, 리전, 테넌트, 모니터링, 흔한 사고, 수신 기능 개요, 안티패턴 |

## 예제 검증

- 02·03의 HCL은 Terraform 1.16.6 + AWS provider 6.x에서 `terraform validate`와 `fmt -check`를 통과했다(2026-10-28).
- 03의 템플릿·IAM 정책 JSON은 파싱을 확인했다.
- CLI 예시(발송, 샌드박스 해제, 알람)는 실제 계정에서 실행하지 않았다. 주소·ARN은 예시 값이다.

## 출처 (1차)

- Amazon SES Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- SES API v2 Reference: https://docs.aws.amazon.com/ses/latest/APIReference-V2/Welcome.html
- AWS CLI `sesv2`: https://docs.aws.amazon.com/cli/latest/reference/sesv2/
- 요금: https://aws.amazon.com/ses/pricing/
- 엔드포인트·쿼터: https://docs.aws.amazon.com/general/latest/gr/ses.html
- Google 발송자 가이드: https://support.google.com/a/answer/81126
- Yahoo 발송자 가이드: https://senders.yahooinc.com/best-practices/

---
*이 문서는 학습·정리 목적의 요약이다. 평판 기준·한도·요금은 자주 바뀌므로 적용 전에 공식 문서를 다시 확인할 것. 실제 주소로 보내는 테스트는 바운스가 계정 평판에 반영되므로 메일박스 시뮬레이터를 쓸 것.*
