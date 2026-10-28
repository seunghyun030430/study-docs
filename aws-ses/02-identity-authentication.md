# Identity와 메일 인증(DKIM·SPF·DMARC)

> 운영에서는 이메일 주소가 아니라 **도메인 identity**를 검증한다. 검증은 **Easy DKIM의 CNAME 3개**로 끝나고, 이 레코드가 곧 DKIM 서명 설정이다. SPF를 From 도메인과 맞추려면(정렬) **custom MAIL FROM 서브도메인**(MX + SPF TXT)을 따로 만든다. 마지막으로 **DMARC**를 `p=none`으로 시작해 `quarantine` → `reject`로 올린다. Gmail·Yahoo는 2024-02부터 이 인증을 요구하고, Gmail은 **2025-11부터 위반 메일을 거부하는 수준으로 강화**했다. identity는 **리전마다** 따로 만든다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)의 용어(identity, 샌드박스, 바운스)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| DKIM | 보내는 서버가 메일에 **전자 서명**을 붙이고, 받는 쪽이 DNS의 공개 키로 위조·변조를 확인하는 방식 |
| Selector | DKIM 공개 키를 찾는 DNS 이름의 앞부분(`<selector>._domainkey.<domain>`) |
| Easy DKIM | SES가 키를 만들고 관리하는 DKIM. DNS에는 CNAME만 추가 |
| BYODKIM | 내가 만든 키 쌍을 SES에 등록하는 DKIM |
| SPF | "이 도메인 메일은 이 서버들만 보낸다"를 DNS TXT로 선언하는 방식. **MAIL FROM(Return-Path) 도메인** 기준으로 검사 |
| MAIL FROM(envelope sender) | 메일 전송 단계의 발신 주소. 바운스가 돌아가는 주소이며, 사용자가 보는 From과 다를 수 있음 |
| DMARC | From 도메인 주인이 "SPF 또는 DKIM이 **From과 정렬**돼 통과해야 한다, 실패하면 이렇게 처리하라"를 선언하는 정책 |
| 정렬(alignment) | SPF·DKIM이 검사한 도메인이 사용자가 보는 From 도메인과 같은(또는 같은 조직 도메인인) 것 |

## Identity

출처: [Creating identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)

| 종류 | 검증 방법 | 쓰임 |
| --- | --- | --- |
| **도메인 identity** | DNS에 DKIM CNAME 추가 | 그 도메인과 서브도메인의 **모든 주소**로 발송 가능. 운영 기본 |
| 이메일 주소 identity | 받은 메일의 링크 클릭(24시간 안에) | 테스트, 도메인 DNS를 못 바꿀 때 |

- **identity는 리전 단위**다. 서울과 도쿄에서 보내려면 각각 만들고 검증한다. 리전당 최대 10,000개.
- 도메인만 검증된 주소에서 configuration set 지정, 발송 위임 같은 고급 기능을 쓰려면 그 주소도 identity로 만들어야 한다.
- 샌드박스에서는 **받는 주소도** 검증된 identity여야 한다([04](./04-reputation-limits.md#샌드박스)).

## DKIM

출처: [DKIM in SES](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim.html), [DkimAttributes](https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_DkimAttributes.html), [SES endpoints: DKIM domains](https://docs.aws.amazon.com/general/latest/gr/ses.html)

### Easy DKIM(권장)

도메인 identity를 만들면 SES가 토큰 3개를 준다. 각각을 CNAME으로 등록한다.

```
<token1>._domainkey.example.com  CNAME  <token1>.dkim.amazonses.com
<token2>._domainkey.example.com  CNAME  <token2>.dkim.amazonses.com
<token3>._domainkey.example.com  CNAME  <token3>.dkim.amazonses.com
```

- CNAME 값의 도메인은 **리전에 따라 다르다.** 서울·도쿄·us-east-1 등 대부분은 `dkim.amazonses.com`이지만, 오사카(ap-northeast-3), 자카르타, 밀라노 등 일부 리전은 `dkim.<region>.amazonses.com`이다. 정확한 값은 API 응답의 `SigningHostedZone`을 쓴다.
- SES는 DNS 레코드를 **최대 72시간** 찾는다. 상태는 `PENDING` → `SUCCESS` / `FAILED` / `TEMPORARY_FAILURE`.
- 키 길이 기본 **2048비트**(1024 선택 가능). 바꾸는 것은 24시간에 한 번까지다.
- CNAME 방식이라 SES가 키를 바꿔도 DNS를 다시 고칠 필요가 없다. 다만 Easy DKIM 키가 **자동으로 얼마나 자주 교체되는지**는 공식 문서에서 찾지 못했다(확인 필요).

```bash
aws sesv2 create-email-identity --email-identity example.com --region ap-northeast-2
# 응답의 DkimAttributes.Tokens 3개와 SigningHostedZone으로 CNAME 생성

aws sesv2 get-email-identity --email-identity example.com --region ap-northeast-2 \
  --query '{Verified:VerifiedForSendingStatus,Dkim:DkimAttributes.Status,Tokens:DkimAttributes.Tokens}'
```

### BYODKIM

키를 직접 관리해야 할 때(여러 발송 서비스가 같은 selector를 써야 하는 경우 등) 쓴다. 키는 1024–2048비트 RSA, PKCS #1 또는 #8, base64 인코딩이다. DNS에는 CNAME이 아니라 **공개 키 TXT**를 직접 넣고, 교체도 직접 한다.

## SPF와 custom MAIL FROM

출처: [Custom MAIL FROM domain](https://docs.aws.amazon.com/ses/latest/dg/mail-from.html)

기본으로 SES 메일의 MAIL FROM은 **`amazonses.com`의 서브도메인**이다. 그래서 SPF는 통과하지만 From(`example.com`)과 **정렬되지 않는다.** DKIM만으로도 DMARC는 통과하지만, SPF까지 정렬하려면 custom MAIL FROM을 설정한다.

```
# MAIL FROM 서브도메인: mail.example.com (보내는 주소나 받는 메일에 쓰지 않는 서브도메인)
mail.example.com  MX   10 feedback-smtp.ap-northeast-2.amazonses.com
mail.example.com  TXT  "v=spf1 include:amazonses.com ~all"
```

| 항목 | 내용 |
| --- | --- |
| 서브도메인 | 검증된 도메인의 서브도메인. **From 주소나 수신용으로 쓰지 않는 것** |
| MX | **정확히 1개**. 여러 개면 설정 실패 |
| 검증 | MX를 72시간 동안 찾고, 못 찾으면 Failed |
| MX 실패 시 동작 | `USE_DEFAULT_VALUE`(amazonses.com으로 발송) / `REJECT_MESSAGE`(발송 거부, `MailFromDomainNotVerified`) |

- DMARC에 `aspf=s`(엄격한 SPF 정렬)를 쓰면 서브도메인이 정렬로 인정되지 않는다. SES 문서는 엄격한 SPF 정책을 쓰지 말라고 한다.
- 루트 도메인에 이미 SPF가 있으면(Google Workspace 등) 그것은 그대로 두고, MAIL FROM 서브도메인에만 SES SPF를 넣는다.

## DMARC

출처: [DMARC in SES](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dmarc.html)

```
_dmarc.example.com  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"
```

> "A message passes DMARC if one or both of the described SPF or DKIM checks pass." — SES DMARC 문서

| 단계 | 정책 | 하는 일 |
| --- | --- | --- |
| 1 | `p=none` | 처리 안 바꿈. `rua` 리포트로 누가 내 도메인으로 보내는지 파악 |
| 2 | `p=quarantine` | 실패 메일을 스팸함으로 |
| 3 | `p=reject` | 실패 메일 거부 |

- SES DKIM은 From 도메인으로 서명하므로 Easy DKIM만 설정해도 **DKIM 정렬로 DMARC를 통과**한다.
- 마케팅 툴, 고객센터 툴 등 **다른 서비스도 같은 도메인으로 메일을 보내는지** 리포트로 확인한 뒤 정책을 올린다. 확인 없이 `reject`로 가면 그 서비스들의 메일이 사라진다.
- 트랜잭셔널 메일과 마케팅 메일을 **서브도메인으로 나누면**(예: `notify.example.com`, `news.example.com`) 한쪽 평판 문제가 다른 쪽으로 번지는 것을 줄인다(업계 관행, 이 레포의 정리).

## Gmail·Yahoo 발송자 요구사항

출처: [Google: Email sender guidelines](https://support.google.com/a/answer/81126), [Google: FAQ](https://support.google.com/a/answer/14229414), [Yahoo: Sender best practices](https://senders.yahooinc.com/best-practices/)

| 요구사항 | 모든 발송자 | 대량 발송자(Gmail: 하루 약 5,000통 이상) |
| --- | --- | --- |
| SPF 또는 DKIM | 필요 | **둘 다** |
| DMARC | — | 필요(`p=none` 가능) |
| From 정렬 | — | SPF 또는 DKIM 도메인과 정렬 |
| 스팸 신고율 | 0.3% 미만(Google 권장 0.1% 미만) | 같음 |
| 원클릭 구독 해지 | — | 마케팅·구독 메일에 필요(RFC 8058) + 본문 링크 |
| DKIM 키 | Google: 1024비트 이상 | 같음 |

- 2024-02-01부터 적용됐다. Gmail은 한 번이라도 대량 발송자 기준에 들면 **계속 대량 발송자로 본다.**
- Google FAQ: **2025-11부터 Gmail이 위반 메일에 대한 집행을 강화**해 일시·영구 거부가 생길 수 있다.
- 비밀번호 재설정 같은 트랜잭셔널 메일은 원클릭 해지 대상이 아니지만, 인증·스팸률 요구는 똑같이 적용된다.
- 스팸 신고율은 **Google Postmaster Tools**에 도메인을 등록해 확인한다.

## Terraform 예시

출처: Terraform Registry [`aws_sesv2_email_identity`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sesv2_email_identity), [`aws_sesv2_email_identity_mail_from_attributes`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sesv2_email_identity_mail_from_attributes)

```hcl
locals {
  domain    = "example.com"
  mail_from = "mail.example.com"
  region    = "ap-northeast-2"
}

data "aws_route53_zone" "main" {
  name = local.domain
}

resource "aws_sesv2_email_identity" "domain" {
  email_identity = local.domain
}

# DKIM CNAME 3개. 서울은 dkim.amazonses.com (일부 리전은 dkim.<region>.amazonses.com)
resource "aws_route53_record" "dkim" {
  count   = 3
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "${aws_sesv2_email_identity.domain.dkim_signing_attributes[0].tokens[count.index]}._domainkey.${local.domain}"
  type    = "CNAME"
  ttl     = 600
  records = ["${aws_sesv2_email_identity.domain.dkim_signing_attributes[0].tokens[count.index]}.dkim.amazonses.com"]
}

resource "aws_sesv2_email_identity_mail_from_attributes" "domain" {
  email_identity         = aws_sesv2_email_identity.domain.email_identity
  mail_from_domain       = local.mail_from
  behavior_on_mx_failure = "USE_DEFAULT_VALUE"
}

resource "aws_route53_record" "mail_from_mx" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = local.mail_from
  type    = "MX"
  ttl     = 600
  records = ["10 feedback-smtp.${local.region}.amazonses.com"]
}

resource "aws_route53_record" "mail_from_spf" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = local.mail_from
  type    = "TXT"
  ttl     = 600
  records = ["v=spf1 include:amazonses.com ~all"]
}

resource "aws_route53_record" "dmarc" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "_dmarc.${local.domain}"
  type    = "TXT"
  ttl     = 600
  records = ["v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"]
}
```

- Registry의 공식 CNAME 예시는 v1 리소스(`aws_ses_domain_dkim`) 기준이다. 위의 v2 토큰(`dkim_signing_attributes[0].tokens`) 사용은 그 예시를 v2로 옮긴 것이다.
- Terraform 리소스는 `SigningHostedZone`을 내보내지 않으므로, `dkim.<region>.amazonses.com`을 쓰는 리전에서는 값을 직접 바꿔야 한다.
- `count`의 토큰 값은 apply 전에는 알 수 없어서, 처음 plan에서 Route 53 레코드가 "known after apply"로 보인다. 정상 동작이다.

## 이슈 / 주의

- **이메일 주소 identity로 운영**: DKIM 정렬이 없어 Gmail·Yahoo 요구를 맞추기 어렵다. 도메인 identity를 쓴다.
- **리전을 옮기면서 identity를 안 만듦**: 새 리전에서 `MessageRejected`(Email address is not verified)가 난다.
- **CNAME 값에 `dkim.amazonses.com`을 하드코딩**: 일부 리전에서 검증이 끝나지 않는다.
- **MAIL FROM 서브도메인에 MX를 여러 개**: 설정이 실패한다.
- **DMARC를 바로 `p=reject`**: 같은 도메인을 쓰는 다른 서비스 메일이 거부된다.
- **루트 도메인 SPF에 `include:amazonses.com`만 추가하고 끝냄**: custom MAIL FROM 없이는 SPF가 From과 정렬되지 않는다(DKIM 정렬로 통과는 가능).
- **`aspf=s`**: SES custom MAIL FROM으로는 SPF 정렬이 안 된다.

## 출처

- https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html
- https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim.html
- https://docs.aws.amazon.com/ses/latest/dg/troubleshoot-dkim.html
- https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_DkimAttributes.html
- https://docs.aws.amazon.com/general/latest/gr/ses.html
- https://docs.aws.amazon.com/cli/latest/reference/sesv2/put-email-identity-dkim-signing-attributes.html
- https://docs.aws.amazon.com/ses/latest/dg/mail-from.html
- https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dmarc.html
- https://support.google.com/a/answer/81126
- https://support.google.com/a/answer/14229414
- https://senders.yahooinc.com/best-practices/
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sesv2_email_identity
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sesv2_email_identity_mail_from_attributes

---
*조사 기준일: 2026-10-28. HCL 예시는 Terraform `validate`로 검사했다(결과는 README 참고). Yahoo 요구사항은 공식 페이지 요약을 바탕으로 정리했다.*
