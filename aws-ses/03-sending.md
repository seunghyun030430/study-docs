# 발송: API·SMTP·템플릿·configuration set

> 애플리케이션에서는 **SES API v2 `SendEmail`**을 쓴다. SMTP는 기존 메일 라이브러리나 서드파티 도구를 붙일 때 쓴다. 모든 발송에 **configuration set**을 지정해, **바운스·불만·배달 이벤트를 SNS·EventBridge 등으로 받는다**. 이벤트를 받지 않으면 바운스를 처리할 수 없고, 그러면 평판이 떨어진다([04](./04-reputation-limits.md)). 메시지는 v2·SMTP 기준 **40 MB, 수신자 50명**까지다. 앱의 IAM 권한은 `ses:FromAddress` 조건으로 **보낼 수 있는 주소를 제한**한다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)·[02](./02-identity-authentication.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Simple / Raw / Template 콘텐츠 | 제목·본문만 주는 방식 / MIME 원문 전체를 주는 방식 / 저장된 템플릿에 값만 주는 방식 |
| MIME | 첨부·HTML·텍스트를 한 메일에 담는 메일 형식 표준 |
| 템플릿 | 제목·본문에 `{{name}}` 같은 자리 표시자를 둔 재사용 메일 틀 |
| Configuration set | 발송에 적용할 규칙 묶음(이벤트 발행, 추적, 전용 IP 풀, suppression 설정) |
| Event destination | configuration set이 이벤트를 보내는 곳(SNS, EventBridge, Firehose, CloudWatch) |
| Open·click 추적 | 본문에 픽셀·리다이렉트 링크를 넣어 열람·클릭을 기록 |
| Sending authorization | 다른 계정·사용자가 내 identity로 보내도록 허락하는 정책 |

## API v2 vs SMTP

출처: [EmailContent](https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_EmailContent.html), [SMTP](https://docs.aws.amazon.com/ses/latest/dg/smtp-connect.html), [SMTP credentials](https://docs.aws.amazon.com/ses/latest/dg/smtp-credentials.html), [Quotas](https://docs.aws.amazon.com/ses/latest/dg/quotas.html)

| 항목 | API v2(`SendEmail`) | SMTP |
| --- | --- | --- |
| 인증 | IAM(role, 임시 자격증명 OK) | **SMTP 자격증명**(IAM 사용자에서 파생, 리전별) |
| 엔드포인트 | `email.<region>.amazonaws.com` | `email-smtp.<region>.amazonaws.com` |
| 포트 | 443 | STARTTLS 25·587·2587, TLS Wrapper 465·2465 |
| 메시지 크기 | 40 MB(base64 인코딩 후) | 40 MB |
| 응답 | `MessageId` 즉시 반환 | SMTP 응답 코드 |
| 맞는 곳 | 새로 만드는 앱(SDK) | 기존 메일 라이브러리, WordPress·Jira 같은 도구 |

- v1 API(`aws ses`)는 메시지 10 MB까지다. 10 MB가 넘는 메시지는 대역폭 제한을 받는다.
- 메시지당 수신자(To + CC + BCC)는 **50명**이다. 많은 사람에게 보낼 때는 **사람마다 따로** 보낸다(BCC로 묶으면 바운스·불만 추적과 개인화가 어렵다).
- SMTP 비밀번호는 시크릿 액세스 키와 다르고, 리전이 들어간 값으로 파생된다. **임시 자격증명으로 만들 수 없으므로** 장기 IAM 사용자가 필요하다. 권한은 `ses:SendRawEmail`만 준다.
- 일부 리전(자카르타, 밀라노, UAE 등)에는 SMTP 엔드포인트가 없다. 서울은 있다(SES 엔드포인트 목록 기준).

```bash
# Simple 콘텐츠 발송 (샌드박스면 받는 주소도 검증돼 있어야 함)
aws sesv2 send-email --region ap-northeast-2 \
  --from-email-address "Example <no-reply@example.com>" \
  --destination 'ToAddresses=user@example.com' \
  --content '{"Simple":{"Subject":{"Data":"비밀번호 재설정","Charset":"UTF-8"},"Body":{"Text":{"Data":"아래 링크로 재설정하세요.","Charset":"UTF-8"}}}}' \
  --configuration-set-name transactional
```

실제 주소로 보내는 명령은 바운스가 생기면 계정 평판에 반영된다. 테스트는 **SES 메일박스 시뮬레이터**(`success@simulator.amazonses.com`, `bounce@simulator.amazonses.com` 등)로 한다(시뮬레이터 주소는 SES 문서 기준, 이번 조사에서 재확인하지 않음).

## 템플릿

출처: [Personalized email](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html), [Advanced templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-advanced.html)

```json
{
  "TemplateName": "password-reset",
  "TemplateContent": {
    "Subject": "{{name}}님, 비밀번호를 재설정하세요",
    "Text": "{{name}}님, 아래 링크는 30분 동안 유효합니다.\n{{resetUrl}}",
    "Html": "<p>{{name}}님,</p><p><a href=\"{{resetUrl}}\">비밀번호 재설정</a></p>"
  }
}
```

```bash
aws sesv2 create-email-template --cli-input-json file://password-reset.json --region ap-northeast-2

aws sesv2 send-email --region ap-northeast-2 \
  --from-email-address no-reply@example.com \
  --destination 'ToAddresses=user@example.com' \
  --content '{"Template":{"TemplateName":"password-reset","TemplateData":"{\"name\":\"홍길동\",\"resetUrl\":\"https://example.com/reset?t=abc\"}"}}' \
  --configuration-set-name transactional
```

| 항목 | 내용 |
| --- | --- |
| 문법 | Handlebars(중첩 속성, `each`, `if`/`else`, partial) |
| 한도 | 리전당 20,000개, 템플릿당 500 KB |
| 인라인 템플릿 | 저장하지 않고 발송 때 내용을 함께 보냄. 20,000개 한도에 안 들어감 |
| `SendBulkEmail` | 템플릿 하나로 **호출당 50개 대상**에게 각자 다른 값으로 발송 |

- **SES는 템플릿 값의 HTML을 이스케이프하지 않는다.** 사용자 입력(이름 등)을 그대로 넣으면 HTML 삽입이 가능하므로 앱에서 이스케이프한다.
- 템플릿 값이 빠지면 렌더링 실패가 난다. `RenderingFailure` 이벤트로 받는다(v2 발송에 이 이벤트가 적용되는 범위는 문서끼리 설명이 달라 확인 필요).
- 템플릿을 SES에 둘지 앱 코드에 둘지는 선택이다. 버전 관리·리뷰·테스트가 쉬운 쪽은 앱 코드(또는 Terraform `aws_sesv2_email_template`)다.

## Configuration set과 이벤트

출처: [Event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html), [Custom tracking domains](https://docs.aws.amazon.com/ses/latest/dg/configure-custom-open-click-domains.html)

| 이벤트 | 뜻 |
| --- | --- |
| Send | SES가 발송 요청을 받아들임 |
| Reject | 바이러스 등으로 SES가 거부 |
| Delivery | 받는 서버가 메일을 받음(메일함 도착은 아님) |
| **Bounce** | 받는 서버가 거부(Permanent: 주소 없음 등 / Transient: 일시적) |
| **Complaint** | 받는 사람이 스팸 신고 |
| DeliveryDelay | 일시적 문제로 배달 지연 |
| RenderingFailure | 템플릿 렌더링 실패 |
| Open / Click | 열람·클릭(추적을 켰을 때) |
| Subscription | 구독 설정 변경(구독 관리 기능) |

| 목적지 | 쓰임 |
| --- | --- |
| **SNS** | Lambda·SQS로 바운스·불만 처리 |
| **EventBridge** | 기본 이벤트 버스로 보내 규칙으로 분기(Terraform 문서: 기본 버스만 지원) |
| Firehose | S3·분석 시스템에 모든 이벤트 저장 |
| CloudWatch | 이벤트를 메트릭으로(차원별 집계) |
| Pinpoint(AWS End User Messaging) | 마케팅 분석 |

- configuration set당 이벤트 목적지는 10개, 리전당 configuration set은 10,000개다.
- identity에 **기본 configuration set**을 지정해 두면, 발송 요청에 지정이 없을 때 그것을 쓴다. 지정을 깜빡한 발송도 이벤트가 남는다.
- Open·click 추적은 링크를 SES 도메인으로 바꾼다. **custom redirect domain**(예: `track.example.com`)을 쓰고 HTTPS가 필요하면 CloudFront를 앞에 둔다. 비밀번호 재설정 링크처럼 민감한 링크는 추적하지 않는 편이 낫다(이 레포의 정리).

```hcl
resource "aws_sesv2_configuration_set" "transactional" {
  configuration_set_name = "transactional"

  delivery_options {
    tls_policy = "REQUIRE" # 받는 서버가 TLS를 지원하지 않으면 보내지 않음
  }

  reputation_options {
    reputation_metrics_enabled = true
  }

  suppression_options {
    suppressed_reasons = ["BOUNCE", "COMPLAINT"]
  }
}

resource "aws_sns_topic" "ses_events" {
  name = "ses-events"
}

resource "aws_sesv2_configuration_set_event_destination" "sns" {
  configuration_set_name = aws_sesv2_configuration_set.transactional.configuration_set_name
  event_destination_name = "bounce-complaint-to-sns"

  event_destination {
    enabled              = true # 기본값 false
    matching_event_types = ["BOUNCE", "COMPLAINT", "DELIVERY", "REJECT", "RENDERING_FAILURE"]

    sns_destination {
      topic_arn = aws_sns_topic.ses_events.arn
    }
  }
}
```

- `event_destination.enabled`는 Terraform 기본값이 **false**다. 빼먹으면 이벤트가 오지 않는다.
- `tls_policy = "REQUIRE"`는 비밀번호 재설정 메일처럼 민감한 내용에 맞지만, TLS를 지원하지 않는 받는 서버에는 배달되지 않는다.
- SNS 토픽 정책에서 SES의 publish를 허용해야 한다(위 예시에는 생략, SES·SNS 문서 확인 필요).

## 구독 해지 헤더

출처: [Subscription management](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html)

- 마케팅·뉴스레터는 `List-Unsubscribe`와 `List-Unsubscribe-Post`(RFC 8058 원클릭) 헤더가 Gmail·Yahoo 대량 발송자 요구사항이다([02](./02-identity-authentication.md#gmailyahoo-발송자-요구사항)).
- SES **contact list + topic**을 쓰고 `ListManagementOptions`를 지정하면 SES가 헤더와 해지 페이지를 자동으로 붙인다. 본문에는 `{{amazonSESUnsubscribeUrl}}`을 넣는다.
- 제약: **수신자가 1명인 메일에만** 헤더가 붙고, 직접 DKIM 서명한 메일에는 쓸 수 없으며, 같은 헤더가 있으면 SES가 덮어쓴다.
- 트랜잭셔널 메일에는 필요하지 않다. 마케팅 메일을 같은 도메인·계정으로 보낸다면 이 기능이나 자체 해지 처리를 갖춘다.

## IAM 권한

출처: [Controlling access](https://docs.aws.amazon.com/ses/latest/dg/control-user-access.html), [Sending authorization](https://docs.aws.amazon.com/ses/latest/dg/sending-authorization.html)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ses:SendEmail", "ses:SendRawEmail"],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "ses:FromAddress": "no-reply@example.com" }
      }
    }
  ]
}
```

| 조건 키 | 제한하는 것 |
| --- | --- |
| `ses:FromAddress` | From 주소 |
| `ses:FromDisplayName` | 보이는 이름 |
| `ses:Recipients` | To·CC·BCC 주소 |
| `ses:FeedbackAddress` | Return-Path |
| `ses:ApiVersion` | API 버전(SMTP는 v2 `SendRawEmail`로 평가) |

- 앱마다 role을 나누고 **보낼 수 있는 주소를 제한**한다. 키가 유출돼도 회사 도메인의 아무 주소로 피싱 메일을 보내지 못하게 하기 위해서다.
- v2 API의 `ses:SendEmail` 권한 하나로 Simple·Raw·Template 발송이 된다. 리소스를 identity ARN으로 좁힐 수도 있다(액션별 리소스 타입은 IAM 서비스 승인 참조에서 확인, [aws-iam/02](../aws-iam/02-policies.md)).
- **다른 계정**이 내 identity로 보내게 하려면 IAM 정책이 아니라 identity의 **sending authorization 정책**을 쓴다.

## 이슈 / 주의

- **configuration set 없이 발송**: 바운스·불만을 알 수 없다. 기본 configuration set을 identity에 지정한다.
- **이벤트 목적지 `enabled` 누락**(Terraform 기본 false).
- **BCC로 대량 발송**: 바운스가 한꺼번에 쌓이고 수신자별 처리가 안 된다.
- **템플릿에 사용자 입력을 그대로**: HTML 삽입. SES는 이스케이프하지 않는다.
- **SMTP 자격증명을 여러 앱이 공유**: 유출 시 범위가 크고 교체가 어렵다. 앱마다 IAM 사용자, 가능하면 API + role.
- **`ses:FromAddress` 제한 없는 권한**: 키 유출이 곧 회사 도메인 피싱이다.
- **재시도 없이 발송**: `Throttling` 오류(발송 속도 초과)는 지수 백오프로 재시도한다([04](./04-reputation-limits.md)).

## 출처

- https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_EmailContent.html
- https://docs.aws.amazon.com/ses/latest/dg/smtp-connect.html
- https://docs.aws.amazon.com/ses/latest/dg/smtp-credentials.html
- https://docs.aws.amazon.com/ses/latest/dg/quotas.html
- https://docs.aws.amazon.com/general/latest/gr/ses.html
- https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html
- https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-advanced.html
- https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html
- https://docs.aws.amazon.com/ses/latest/dg/configure-custom-open-click-domains.html
- https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html
- https://docs.aws.amazon.com/ses/latest/dg/control-user-access.html
- https://docs.aws.amazon.com/ses/latest/dg/sending-authorization.html
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sesv2_configuration_set
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sesv2_configuration_set_event_destination

---
*조사 기준일: 2026-10-28. HCL은 Terraform `validate`, JSON은 파싱 검사를 했다(결과는 README 참고). CLI 발송 예시는 실제로 실행하지 않았다.*
