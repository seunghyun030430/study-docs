# Amazon SES 개요

> Amazon SES(Simple Email Service)는 앱에서 메일을 보내는(그리고 받는) AWS 관리형 서비스다. 시작 순서는 정해져 있다.
> 1. **도메인 identity를 검증**하고 DKIM·SPF·DMARC를 설정한다.
> 2. **샌드박스 해제**를 요청한다. 리전마다, 처음에는 하루 200통·초당 1통만 보낼 수 있다.
> 3. configuration set으로 **바운스·불만 이벤트를 받아 처리**한다.
>
> 바운스율 5%·불만율 0.1%를 넘으면 계정이 검토 대상이 되고, 더 오르면 **발송이 정지될 수 있다.** 그래서 SES 운영의 핵심은 "보내기"보다 **"평판 지키기"**다. 요금은 1,000통당 약 $0.10부터이고, 2026-07부터는 요금제(plan)를 고를 수 있다.

## 1단계: 메일은 어떻게 전달되나

앱이 메일을 보낼 때 실제로 일어나는 일은 이렇다.

```
앱 ──▶ 발송 서버(SES) ──인터넷(SMTP)──▶ 받는 쪽 메일 서버(Gmail, Naver 등) ──▶ 받은편지함 / 스팸함
```

- 받는 쪽 서버는 "이 메일이 정말 그 도메인에서 왔나", "이 발송 서버는 믿을 만한가"를 따져 **받을지, 스팸함에 넣을지, 거부할지** 정한다.
- 메일 서버를 직접 운영하면 IP 평판 관리, 인증 설정, 스팸 차단 대응을 모두 해야 한다. SES는 이 **발송 서버 역할**을 대신한다.

## 2단계: 트랜잭셔널 메일과 마케팅 메일

| 종류 | 예시 | 특징 |
| --- | --- | --- |
| **트랜잭셔널** | 가입 인증, 비밀번호 재설정, 결제 영수증, 알림 | 사용자의 행동으로 1:1 발송. 늦거나 스팸함에 가면 서비스 장애 |
| 마케팅 | 뉴스레터, 프로모션 | 1:N 대량 발송. 구독 해지 기능이 법·메일함 사업자 요구사항 |

이 문서 모음은 **트랜잭셔널 발송**을 중심으로 다룬다. 두 종류를 같은 도메인·IP로 보내면 마케팅 메일의 스팸 신고가 트랜잭셔널 메일 도달률까지 떨어뜨릴 수 있으므로 나누는 것이 좋다([02](./02-identity-authentication.md#dmarc)).

## 3단계: Identity — 누가 보내는가

SES는 **검증한 주소·도메인으로만** 보내게 한다. 이 "보내는 주체"를 **identity**라고 한다.

| identity | 검증 방법 |
| --- | --- |
| 이메일 주소(`no-reply@example.com`) | 그 주소로 온 확인 링크 클릭 |
| **도메인**(`example.com`) | 도메인 DNS에 SES가 준 레코드 추가 → 그 도메인의 모든 주소로 발송 가능 |

identity는 **리전마다** 따로 만든다. 서울(ap-northeast-2)에서 검증한 도메인은 도쿄에서 쓸 수 없다.

## 4단계: 메일 인증 — 위조가 아님을 증명하기

누구나 메일 From에 `ceo@example.com`이라고 적을 수 있다. 그래서 받는 서버는 세 가지 방법으로 진짜인지 확인한다.

| 방법 | 한 줄 설명 |
| --- | --- |
| **SPF** | "example.com 메일은 이 서버들만 보낸다"를 DNS에 적어 둠 |
| **DKIM** | 보내는 서버가 메일에 전자 서명을 붙이고, 받는 쪽이 DNS의 공개 키로 확인 |
| **DMARC** | "SPF나 DKIM이 From 도메인과 맞아야 통과, 실패하면 거부하라"는 도메인 주인의 정책 |

Gmail·Yahoo는 2024-02부터 이 인증을 요구하고, 인증이 없는 메일은 스팸함에 가거나 거부된다. 설정 방법은 [02](./02-identity-authentication.md)에서 다룬다.

## 5단계: 샌드박스 — 처음엔 제한된 상태

새 계정은 리전마다 **샌드박스(sandbox)** 상태로 시작한다.

| 샌드박스 | 프로덕션 |
| --- | --- |
| **검증된 주소에만** 발송(또는 메일박스 시뮬레이터) | 아무 주소로나 발송 |
| 24시간 200통, 초당 1통 | 계정마다 정해진 한도(사용하면서 늘어남) |

실제 사용자에게 보내려면 **프로덕션 액세스를 요청**해야 한다. AWS 지원팀이 96시간 안에 처음 답한다. 요청 방법은 [04](./04-reputation-limits.md)에서 다룬다.

## 6단계: 바운스와 불만 — 평판을 깎는 두 가지

| 용어 | 뜻 | 예 |
| --- | --- | --- |
| **바운스(bounce)** | 받는 서버가 메일을 거부하고 돌려보냄 | 없는 주소(hard bounce), 메일함 가득 참(soft bounce) |
| **불만(complaint)** | 받는 사람이 "스팸 신고"를 누름 | 가입한 적 없는 사람에게 보낸 메일 |

SES는 이 비율로 계정 평판을 본다.

| 지표 | 권장 | 검토(under review) | 정지 가능(paused) |
| --- | --- | --- | --- |
| 바운스율 | 2% 미만 | 5% 이상 | 10% 이상 |
| 불만율 | 0.1% 미만 | 0.1% 이상 | 0.5% 이상 |

그래서 **바운스·불만 이벤트를 받아서 그 주소로 다시 보내지 않는 처리**가 필수다. SES의 **suppression list**가 이를 자동으로 돕는다([04](./04-reputation-limits.md)).

## 7단계: 보내는 방법

| 방법 | 쓰임 |
| --- | --- |
| **SES API v2**(`SendEmail`) | 앱에서 SDK로. 권장 |
| SMTP | 기존 메일 라이브러리·도구 연결 |

발송할 때 **configuration set**을 지정하면, 바운스·불만·배달 같은 **이벤트**를 SNS·EventBridge 등으로 받을 수 있다. 자세한 내용은 [03](./03-sending.md)에 있다.

## 8단계: SES 기능 지도

| 영역 | 기능 | 문서 |
| --- | --- | --- |
| 인증 | 도메인 identity, Easy DKIM, custom MAIL FROM, DMARC | [02](./02-identity-authentication.md) |
| 발송 | API v2, SMTP, 템플릿, 대량 발송(`SendBulkEmail`), 구독 관리 | [03](./03-sending.md) |
| 이벤트 | configuration set, 이벤트 목적지(SNS, EventBridge, Firehose, CloudWatch) | [03](./03-sending.md) |
| 평판 | 샌드박스, 발송 한도, 평판 메트릭, suppression list | [04](./04-reputation-limits.md) |
| 도달률 | Virtual Deliverability Manager(VDM), 전용 IP, 테넌트 | [04](./04-reputation-limits.md), [05](./05-cost-operations-issues.md) |
| 수신 | 수신 규칙(S3·Lambda·SNS), Mail Manager | [05](./05-cost-operations-issues.md#수신-기능-개요) |

## 9단계: 요금 개요

출처: [SES pricing](https://aws.amazon.com/ses/pricing/), [Pricing plans](https://docs.aws.amazon.com/ses/latest/dg/pricing-plans.html)

| 방식 | 발송 1,000통당 | 고정비 |
| --- | --- | --- |
| 기존(à la carte) | $0.10 | 없음 |
| Essentials 요금제 | $0.16(월 1,000만 통까지) | 없음 |
| Pro / Enterprise | $0.22 / $0.23(월 1,000만 통까지) | 리전당 월 $105 / $500 |

첨부는 GB당 $0.12가 더 붙는다. 2026-07-21에 요금제가 도입됐다. 이후 새로 시작하는 계정·리전은 Essentials로 시작한다는 요금 페이지 주석이 있다. 자세한 비교는 [05](./05-cost-operations-issues.md#요금)에 있다.

## 용어 정리

| 용어 | 뜻 | 처음 나온 곳 |
| --- | --- | --- |
| SES | AWS의 메일 발송·수신 서비스 | 요약 |
| 트랜잭셔널 메일 | 사용자 행동으로 1:1 발송하는 메일 | 2단계 |
| Identity | SES에서 검증한 보내는 주소·도메인 | 3단계 |
| SPF / DKIM / DMARC | 메일 위조를 막는 세 가지 인증 방식 | 4단계 |
| 샌드박스 | 검증된 주소에만 소량 보낼 수 있는 초기 상태 | 5단계 |
| 프로덕션 액세스 | 샌드박스를 벗어난 상태 | 5단계 |
| 바운스(hard / soft) | 받는 서버의 거부(영구 / 일시) | 6단계 |
| 불만(complaint) | 받는 사람의 스팸 신고 | 6단계 |
| Suppression list | 바운스·불만이 난 주소로 다시 보내지 않게 막는 목록 | 6단계 |
| Configuration set | 발송에 적용하는 이벤트·추적·IP 설정 묶음 | 7단계 |
| 이벤트 목적지 | 발송 이벤트를 받는 곳(SNS 등) | 7단계 |
| VDM | 도달률 대시보드·조언 기능(Virtual Deliverability Manager) | 8단계 |
| 전용 IP | 내 계정만 쓰는 발송 IP | 8단계 |

## 출처

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html
- https://docs.aws.amazon.com/ses/latest/dg/email-authentication-methods.html
- https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html
- https://docs.aws.amazon.com/ses/latest/dg/faqs-enforcement.html
- https://docs.aws.amazon.com/ses/latest/dg/regions.html
- https://aws.amazon.com/ses/pricing/
- https://docs.aws.amazon.com/ses/latest/dg/pricing-plans.html

---
*조사 기준일: 2026-10-28. 요금은 요금 페이지의 us-east-1 표시 금액이다.*
