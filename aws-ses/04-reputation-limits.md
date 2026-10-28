# 샌드박스·한도·평판·바운스 처리

> 샌드박스는 **리전마다** 해제해야 하고, 요청하면 지원팀이 **96시간 안에** 처음 답한다. 발송 한도는 **24시간 발송량과 초당 발송 속도** 둘이며, 메시지가 아니라 **수신자 수**로 센다. 평판 기준은 **바운스율 5%(검토)·10%(정지 가능), 불만율 0.1%(검토)·0.5%(정지 가능)**다. 계정이 정지돼도 **EventBridge 이벤트가 오지 않으므로** `GetAccount`의 `EnforcementStatus`와 CloudWatch 알람으로 감시한다. 계정 단위 **suppression list**를 켜 두고, 바운스·불만 이벤트로 **앱의 사용자 DB도 갱신**한다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[03](./03-sending.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Sending quota | 24시간(롤링) 동안 보낼 수 있는 최대 수신자 수 |
| Sending rate | 초당 받아들이는 최대 메일 수 |
| Enforcement status | 계정 평판 상태(`HEALTHY` / `PROBATION` / `SHUTDOWN`) |
| Hard / soft bounce | 영구 실패(주소 없음) / 일시 실패(메일함 가득 참 등) |
| FBL(Feedback loop) | 메일함 사업자가 스팸 신고를 발송자에게 알려 주는 장치 |
| Account-level suppression list | 내 계정에서 hard bounce·불만이 난 주소로 보내지 않게 막는 목록 |
| Global suppression list | SES 전체가 공유하는, 최근 hard bounce가 난 주소 목록 |
| Mailbox simulator | 바운스·불만 등을 흉내 내는 SES 테스트 주소 |

## 샌드박스

출처: [Request production access](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html)

| 제한 | 내용 |
| --- | --- |
| 수신자 | **검증된 주소·도메인** 또는 메일박스 시뮬레이터만 |
| 한도 | 24시간 200통, 초당 1통 |
| 범위 | **리전마다** 따로. 서울을 해제해도 도쿄는 샌드박스 |

```bash
aws sesv2 put-account-details --region ap-northeast-2 \
  --production-access-enabled \
  --mail-type TRANSACTIONAL \
  --website-url https://example.com \
  --additional-contact-email-addresses ops@example.com \
  --contact-language EN
```

- 요청에는 메일 종류(트랜잭셔널/마케팅), 웹사이트, 연락처가 들어가고, **원하는 사람에게만 보내며 바운스·불만 처리 절차가 있다**는 확인이 필요하다.
- 지원팀은 **96시간 안에** 처음 답한다. 정보가 부족하면 더 걸린다. 도메인 identity를 먼저 검증해 두면 승인이 빨라진다고 문서에 적혀 있다.
- 해제 후에도 From·Return-Path에 쓰는 주소는 계속 검증된 identity여야 한다.
- **출시 일정에 맞춰 미리 요청한다.** 해제를 기다리느라 가입 메일이 안 나가는 일이 흔하다.

## 발송 한도

출처: [Sending quotas](https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas.html), [Request increase](https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas-request-increase.html), [GetAccount](https://docs.aws.amazon.com/cli/latest/reference/sesv2/get-account.html)

```bash
aws sesv2 get-account --region ap-northeast-2 \
  --query '{Prod:ProductionAccessEnabled,Status:EnforcementStatus,Quota:SendQuota}'
# SendQuota: Max24HourSend, MaxSendRate, SentLast24Hours (Max24HourSend가 -1이면 무제한)
```

| 항목 | 내용 |
| --- | --- |
| 세는 단위 | **수신자 수**. 수신자 10명인 메일 1통 = 10 |
| 24시간 | 롤링 윈도(자정 리셋 아님) |
| 초당 속도 | 짧은 순간은 넘어도 되지만 계속 넘으면 거부 |
| 자동 증가 | 실제 운영 메일을 한도 가까이 보내고, 바운스·불만이 낮으면 늘려 줄 수 있음 |
| 수동 증가 | Service Quotas 콘솔에서 요청. 처리 목표 최대 24시간 |

### 한도 초과(throttling)

- API는 `ThrottlingException`("Maximum sending rate exceeded" / "Daily message quota exceeded"), SMTP는 `454 Throttling failure`다.
- 거부된 메일은 **대기열에 쌓이지 않고 버려진다.** 앱이 다시 보내야 한다.
- AWS 문서는 최대 10분 기다린 뒤 재시도하라고 한다. 실무에서는 `MaxSendRate`에 맞춰 **앱 쪽에서 속도를 제한**하고(SQS + 소비자 등), 지수 백오프와 지터로 재시도한다(일반 관행).
- **재시도는 중복 발송을 만들 수 있다.** 요청이 타임아웃돼도 SES가 이미 받아들였을 수 있다. 메시지 태그로 이벤트를 맞추고, 중복이 문제가 되는 메일(결제 영수증 등)은 앱에서 멱등 처리한다.

## 평판과 계정 정지

출처: [Enforcement FAQ](https://docs.aws.amazon.com/ses/latest/dg/faqs-enforcement.html), [Reputation alarms](https://docs.aws.amazon.com/ses/latest/dg/reputationdashboard-cloudwatch-alarm.html)

| 지표 | 권장 | 검토 | 정지 가능 |
| --- | --- | --- | --- |
| 바운스율 | **2% 미만** | 5% 이상 | 10% 이상 |
| 불만율 | **0.1% 미만** | 0.1% 이상 | 0.5% 이상 |

- 바운스율은 **검증하지 않은 도메인으로 간 hard bounce**만 센다. 불만율은 **SES에 신고 정보를 주는 사업자**로 간 메일만 센다.
- 비율은 고정 기간이 아니라 "대표성 있는 발송량" 기준이라, `GetSendStatistics`로 그대로 재현할 수 없다.
- **검토(under review)**: 계속 보낼 수 있다. 기간 안에 고치지 못하면 정지될 수 있다.
- **정지(paused)**: 발송이 "Account is paused"로 실패한다. 심각하거나 반복되면 검토 없이 바로 정지될 수도 있다.
- 복구: AWS가 연 지원 케이스에 **원인, 고친 내용, 재발 방지책**을 답한다.

### 감시

> "SES does not emit an Amazon EventBridge event when it pauses or reinstates sending for your entire account." — SES Enforcement FAQ

```bash
# AWS 권장 임계값: 바운스 5%, 불만 0.1%
aws cloudwatch put-metric-alarm --region ap-northeast-2 \
  --alarm-name ses-bounce-rate --namespace AWS/SES --metric-name Reputation.BounceRate \
  --statistic Average --period 3600 --evaluation-periods 1 \
  --threshold 0.05 --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data ignore \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:oncall

aws cloudwatch put-metric-alarm --region ap-northeast-2 \
  --alarm-name ses-complaint-rate --namespace AWS/SES --metric-name Reputation.ComplaintRate \
  --statistic Average --period 3600 --evaluation-periods 1 \
  --threshold 0.001 --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data ignore \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:oncall
```

- AWS 권장값(5%, 0.1%)은 이미 "검토" 기준이다. 그보다 앞선 **경고 알람**(예: 바운스 2~3%)을 하나 더 두는 편이 안전하다(이 레포의 제안).
- 정지 여부는 `GetAccount`의 `EnforcementStatus`를 주기적으로 확인하거나 **AWS Health** 이벤트로 받는다. 테넌트 단위 정지는 EventBridge 이벤트가 있다([05](./05-cost-operations-issues.md#테넌트)).
- 평판 메트릭은 configuration set의 `reputation_metrics_enabled`로 configuration set별로도 볼 수 있다.

## Suppression list

출처: [Suppression list](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)

| 목록 | 범위 | 동작 |
| --- | --- | --- |
| **계정 단위** | 계정·리전 | 목록에 있는 주소로 보내면 SES가 **받아들이지만 실제로 보내지 않음**. 2019-11-25 이후 계정은 BOUNCE·COMPLAINT 기본 켜짐 |
| Global(공유) | SES 전체 | 계정 목록에 없으면 **SES가 그래도 배달을 시도**하고, 바운스되면 바운스율에 반영 |
| configuration set 단위 | configuration set | 계정 목록 사용 / 끄기 / 자체 사유 지정 |
| 테넌트 단위 | 테넌트 | 테넌트별 목록 |

- 계정 목록에는 **hard bounce만** 들어간다(soft bounce는 안 들어감).
- 목록 때문에 막힌 발송은 **평판 비율에는 안 들어가지만 24시간 한도는 쓴다.** 알림에는 `OnAccountSuppressionList` 하위 유형으로 나온다.
- 주소는 **직접 지울 때까지** 남는다(정지된 계정은 90일 후 자동 삭제).
- **Gmail은 SES에 불만 정보를 주지 않는다.** Gmail 사용자의 스팸 신고는 suppression list에 들어가지 않으므로 Google Postmaster Tools로 따로 본다.

```bash
# 오래된 계정에서 켜기
aws sesv2 put-account-suppression-attributes --suppressed-reasons BOUNCE COMPLAINT --region ap-northeast-2

# 사용자가 주소를 고쳤다고 할 때 목록에서 빼기 (다시 바운스되면 평판에 반영)
aws sesv2 delete-suppressed-destination --email-address user@example.com --region ap-northeast-2

aws sesv2 list-suppressed-destinations --reasons BOUNCE --region ap-northeast-2
```

## 바운스·불만 처리 파이프라인

출처: [Notification contents](https://docs.aws.amazon.com/ses/latest/dg/notification-contents.html), [Email feedback](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications-email.html)

```
SES ──(configuration set 이벤트)──▶ SNS ──▶ SQS ──▶ Lambda / 워커
                                                    ├─ Permanent bounce → 사용자 DB: email_status = invalid, 발송 중지
                                                    ├─ Complaint        → 마케팅 수신 동의 해제
                                                    └─ Transient        → 로그만 (SES가 재시도함)
```

| bounceType | 하위 유형 예 | 앱 처리 |
| --- | --- | --- |
| **Permanent** | General, NoEmail, Suppressed, OnAccountSuppressionList | 그 주소로 다시 보내지 않기, 사용자에게 주소 확인 요청 |
| Transient | MailboxFull, MessageTooLarge, ContentRejected | SES가 일정 시간 재시도. 반복되면 확인 |
| Undetermined | — | 로그 |

- suppression list가 있어도 **앱 DB를 갱신**해야 한다. 그래야 다른 채널(알림톡 등)로 안내하거나, 가입 단계에서 주소를 다시 받을 수 있다.
- 불만 알림에서 사업자 대부분은 신고자 주소를 지운다. SES는 원래 메시지의 수신자로 추정한다.
- **이메일 피드백 전달은 기본 켜짐**이다. SNS·이벤트로 받는다면 끄되, 피드백 수단이 하나도 없으면 SES는 이메일로 계속 보낸다.
- SNS 토픽은 SES와 **같은 리전**이어야 한다.
- 가입 단계에서 **이메일 인증(더블 옵트인)**을 하면 오타 주소로 인한 hard bounce가 크게 준다(일반 관행).

## Virtual Deliverability Manager(VDM)

출처: [VDM](https://docs.aws.amazon.com/ses/latest/dg/vdm.html), [SES deliverability](https://docs.aws.amazon.com/ses/latest/dg/vdm-ses-deliverability.html)

| 구성 | 하는 일 | 요금(à la carte) |
| --- | --- | --- |
| SES deliverability | 계정·ISP·identity·configuration set별 도달률 대시보드, 설정 문제 **Advisor**, engagement 추적, optimized shared delivery | 1,000통당 $0.07(월 1,000만 통까지) |
| Global deliverability | 캠페인 분석, 받은편지함 배치 테스트, 블록리스트 감시 | 리전당 월 $1,250 |

- Advisor는 DKIM·SPF·DMARC 같은 인증 레코드 문제를 알려 준다. 초기 설정 점검에 유용하다.
- optimized shared delivery가 정확히 어떻게 동작하는지는 문서에 자세히 없다(확인 필요).
- 요금제(Pro 등)에 따라 포함 범위가 달라진다([05](./05-cost-operations-issues.md#요금)).

## 이슈 / 주의

- **출시 직전에 샌드박스 해제 요청**: 최대 96시간 이상 걸릴 수 있다.
- **리전을 추가하면서 샌드박스를 잊음**: 새 리전은 다시 샌드박스다.
- **수신자 수로 세는 한도를 메시지 수로 착각**: CC가 많으면 한도가 빨리 찬다.
- **throttling을 재시도하지 않음**: 거부된 메일은 사라진다.
- **오래된 주소 목록으로 발송**: 바운스율이 바로 5%를 넘는다. 오래 쓰지 않은 목록은 재확인 후 보낸다.
- **계정 정지를 이벤트로만 기다림**: 계정 단위 정지는 EventBridge 이벤트가 없다.
- **suppression list만 믿고 앱 DB는 그대로**: 사용자는 왜 메일이 안 오는지 모른다.
- **Gmail 불만을 SES 지표로만 판단**: Gmail 신고는 SES에 오지 않는다.

## 출처

- https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html
- https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas.html
- https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas-request-increase.html
- https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas-errors.html
- https://docs.aws.amazon.com/cli/latest/reference/sesv2/get-account.html
- https://docs.aws.amazon.com/general/latest/gr/ses.html
- https://docs.aws.amazon.com/ses/latest/dg/faqs-enforcement.html
- https://docs.aws.amazon.com/ses/latest/dg/reputationdashboard-cloudwatch-alarm.html
- https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html
- https://docs.aws.amazon.com/ses/latest/dg/notification-contents.html
- https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications-email.html
- https://docs.aws.amazon.com/ses/latest/dg/troubleshoot-error-messages.html
- https://docs.aws.amazon.com/ses/latest/dg/vdm.html
- https://docs.aws.amazon.com/ses/latest/dg/vdm-ses-deliverability.html
- https://aws.amazon.com/ses/pricing/

---
*조사 기준일: 2026-10-28. CLI 예시는 실제 계정에서 실행하지 않았다.*
