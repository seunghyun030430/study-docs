# Parameter Store

> 설정값과 시크릿을 계층형 이름(`/app/prod/db/host`)으로 저장하는 키-값 저장소다. Standard 티어는 무료(4 KB, 계정·리전당 1만 개)이고, 그 이상이 필요하거나 만료 정책·계정 간 공유를 쓰려면 Advanced 티어(유료)를 쓴다. **자동 로테이션이 필요한 자격증명은 Secrets Manager가 맞다.**

## 개요 / 정의

Parameter Store는 Systems Manager의 Application tools에 속한다. 애플리케이션 설정, AMI ID, DB 엔드포인트, 비밀번호 같은 값을 저장하고 IAM으로 접근을 제어한다. 모든 변경은 버전으로 남는다.

## 파라미터 타입

출처: [What is a parameter](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-a-parameter.html)

| 타입 | 설명 | 비고 |
| --- | --- | --- |
| `String` | 일반 문자열 | data type `aws:ec2:image`를 지정하면 AMI ID 형식을 검증 |
| `StringList` | 쉼표로 구분한 값 목록 | `a,b,c` |
| `SecureString` | KMS로 **값만** 암호화 | 이름·설명 등 메타데이터는 암호화되지 않음 |

이름 규칙:

- 대소문자를 구분하며, `a-zA-Z0-9_.-`와 `/`만 쓸 수 있다.
- ARN 접두사를 포함해 최대 1011자, 계층은 최대 15단계다.
- `aws`, `ssm`으로 시작할 수 없다(대소문자 무관).

## 티어 비교

출처: [Advanced parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-advanced-parameters.html), [Pricing](https://aws.amazon.com/systems-manager/pricing/)

| 항목 | Standard | Advanced |
| --- | --- | --- |
| 계정·리전당 최대 개수 | 10,000 | 100,000 |
| 값 최대 크기 | 4 KB | 8 KB |
| Parameter policies | 미지원 | 지원(파라미터당 최대 10개) |
| 계정 간 공유(AWS RAM) | 미지원 | 지원 |
| 저장 요금 | 없음 | 파라미터당 월 $0.05(시간 단위 비례) |
| API 요금(기본 처리량) | 없음 | 10,000건당 $0.05 |
| API 요금(높은 처리량) | 10,000건당 $0.05 | 10,000건당 $0.05 |

- Standard → Advanced 업그레이드는 가능하지만 **다운그레이드는 불가**하다. 되돌리려면 삭제 후 Standard로 다시 만들어야 한다.
- 기본 티어는 `Standard`(기본값), `Advanced`, `Intelligent-Tiering` 중에서 고른다. Intelligent-Tiering은 평소 Standard로 만들다가, 값이 4 KB를 넘거나 정책을 붙이거나 1만 개 한도에 닿으면 자동으로 Advanced로 만든다. ([Default tier](https://docs.aws.amazon.com/systems-manager/latest/userguide/ps-default-tier.html))
- API 요금의 "interaction"은 **요청 1건 × 파라미터 1개** 단위다. 파라미터 10개를 반환하는 Get 1회는 10건으로 센다.

### Parameter policies (Advanced 전용)

출처: [Parameter policies](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-policies.html)

| 정책 | 동작 |
| --- | --- |
| `Expiration` | 지정 시각에 파라미터 삭제 |
| `ExpirationNotification` | 만료 전(`Before`, 단위 `Days`/`Hours`) EventBridge 이벤트 발행 |
| `NoChangeNotification` | 일정 기간(`After`) 변경이 없으면 EventBridge 이벤트 발행 |

정책은 비동기 주기 스캔으로 적용되므로 정확한 시각에 실행된다고 가정하면 안 된다. 새 정책을 추가하면 기존 정책을 **덮어쓰고**, `[{}]`을 넣으면 모든 정책이 제거된다.

## 처리량(TPS)

출처: [Parameter Store throughput](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-throughput.html)

| API | 기본 | Higher throughput 활성화 시 |
| --- | --- | --- |
| `GetParameter` | 40 TPS(아래 3개 API 합산 공유) | 10,000 TPS |
| `GetParameters` | ↑ 공유 | 1,000 TPS |
| `GetParametersByPath` | ↑ 공유 | 100 TPS |
| `PutParameter`, `DescribeParameters`, `GetParameterHistory`, `Label/UnlabelParameterVersion` | 3 TPS | 10 TPS |
| `DeleteParameter(s)` | 3 TPS | 5 TPS |

- SecureString은 KMS 요청 한도에도 걸릴 수 있다. ([Quotas](https://docs.aws.amazon.com/general/latest/gr/ssm.html#parameter-store))
- Higher throughput은 계정·리전 단위 설정이고 언제든 끌 수 있다. 켜면 Standard 파라미터 API 호출에도 요금이 붙는다.

```bash
aws ssm update-service-setting \
  --setting-id arn:aws:ssm:ap-northeast-2:111122223333:servicesetting/ssm/parameter-store/high-throughput-enabled \
  --setting-value true
```

## SecureString과 KMS

출처: [How Parameter Store uses AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/services-parameter-store.html)

| 항목 | 내용 |
| --- | --- |
| 기본 키 | AWS managed key `alias/aws/ssm`. 키 정책을 직접 수정할 수 없음 |
| 사용자 키 | `--key-id`로 customer managed key 지정(대칭 키만 지원) |
| Standard SecureString | KMS `Encrypt`로 직접 암호화 |
| Advanced SecureString | AWS Encryption SDK로 envelope encryption(`GenerateDataKey`) |
| 필요한 권한 | 쓰기: `kms:Encrypt`(Standard) / `kms:GenerateDataKey`(Advanced), 읽기: `kms:Decrypt` |
| Encryption context | `PARAMETER_ARN` |

SecureString 자체에는 Parameter Store 요금이 없지만 KMS 요금은 별도로 발생할 수 있다.

## 버전과 라벨

출처: [Parameter labels](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-labels.html), [Quotas](https://docs.aws.amazon.com/general/latest/gr/ssm.html#parameter-store)

- 값을 바꿀 때마다 새 버전이 생기고, 최대 100개까지 보관된다.
- `이름:버전` 또는 `이름:라벨` 형식으로 특정 버전을 조회한다. 없는 버전을 요청하면 최신 버전으로 대체되지 않고 **실패한다**.
- 라벨 규칙:
  - 한 버전에 최대 10개, 라벨 하나당 최대 100자.
  - 숫자, `aws`, `ssm`으로 시작할 수 없다.
  - 같은 파라미터의 두 버전에 같은 라벨을 동시에 붙일 수 없다. 대신 라벨을 다른 버전으로 옮길 수 있다.
  - 파라미터를 만들 때는 라벨을 붙일 수 없다.

## 사용법 / 예시

```bash
# 생성(String / SecureString)
aws ssm put-parameter --name /myapp/prod/db/host --type String --value "db.internal"
aws ssm put-parameter --name /myapp/prod/db/password --type SecureString \
  --value 'S3cr3t!' --key-id alias/myapp-prod

# 값 수정(새 버전 생성)
aws ssm put-parameter --name /myapp/prod/db/host --type String --value "db2.internal" --overwrite

# 조회
aws ssm get-parameter --name /myapp/prod/db/password --with-decryption \
  --query Parameter.Value --output text

# 계층 단위 조회
aws ssm get-parameters-by-path --path /myapp/prod --recursive --with-decryption

# 특정 버전 / 라벨
aws ssm get-parameter --name /myapp/prod/db/host:1
aws ssm label-parameter-version --name /myapp/prod/db/host --parameter-version 2 --labels stable
aws ssm get-parameter --name /myapp/prod/db/host:stable

# 이력
aws ssm get-parameter-history --name /myapp/prod/db/host
```

### Public parameters — 최신 AMI ID 조회

AWS가 관리하는 공개 파라미터에서 최신 AMI ID를 조회할 수 있다. ([Public parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-public-parameters-ami.html))

```bash
aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text
```

### Secrets Manager 시크릿 참조

`/aws/reference/secretsmanager/<secret-name>` 형식의 이름으로 Parameter Store API에서 Secrets Manager 시크릿을 읽을 수 있다. 다만 `GetParameter`/`GetParameters`만 지원하며 `GetParametersByPath`는 쓸 수 없다. 버전 관리는 Secrets Manager 쪽 규칙을 따른다. ([Referencing Secrets Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/integration-ps-secretsmanager.html))

```bash
aws ssm get-parameter --name /aws/reference/secretsmanager/myapp-db --with-decryption
```

### 계정 간 공유

출처: [Shared parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-shared-parameters.html)

- AWS RAM으로 공유하며, **Advanced 파라미터만** 공유할 수 있다.
- SecureString을 공유하려면 customer managed KMS 키를 써야 하고, 그 키도 따로 공유해야 한다(`aws/ssm` 키로 암호화한 값은 공유 불가).
- 공유받은 계정은 읽기만 할 수 있고, 조회할 때 전체 ARN을 써야 한다.
- 지원: CloudFormation 템플릿 파라미터, EC2 launch template.
- 미지원: CloudFormation dynamic reference, ECS secrets, CodeBuild 환경변수.
- 요금: 저장 비용은 소유 계정이, API 호출 비용은 호출한 계정이 낸다.

## Parameter Store vs Secrets Manager

| 항목 | Parameter Store | Secrets Manager |
| --- | --- | --- |
| 주 용도 | 설정값 + 단순 시크릿 | DB 자격증명·API 키 등 로테이션이 필요한 시크릿 |
| 자동 로테이션 | 없음(만료·알림 정책만) | managed rotation / Lambda rotation 지원 |
| 값 최대 크기 | 4 KB / 8 KB | 64 KB |
| 저장 비용 | Standard 무료, Advanced 월 $0.05 | 시크릿당 월 $0.40 |
| API 비용 | Standard 기본 처리량 무료 | 10,000건당 $0.05 |
| 계정 간 공유 | Advanced + AWS RAM | 리소스 정책으로 지원 |
| 버전 | 최대 100개 | 최대 100개(staging label 기반) |

> AWS는 "automatic rotation, cross-account access, or fine-grained audit logging"이 필요하면 Secrets Manager를 권장한다. — [Parameter policies](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-policies.html)

Secrets Manager 수치 출처: [Secrets Manager pricing](https://aws.amazon.com/secrets-manager/pricing/), [Secrets Manager quotas](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_limits.html). 요금 페이지 표가 렌더링되지 않아 계산 예시로 값을 확인했다(확인 필요).

## 이슈 / 안티패턴 / 주의

- **경로 권한 함정**: `GetParametersByPath`는 요청한 **경로**를 기준으로 권한을 검사한다. `/a`에 권한이 있으면 `/a/b`를 명시적으로 Deny했더라도 `/a/b`가 반환된다. ([What is a parameter](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-a-parameter.html))
- **40 TPS 기본 한도**: 컨테이너나 Lambda가 시작할 때마다 `GetParameter`를 부르면 쉽게 `ThrottlingException`이 난다. 앱에서 값을 캐시하거나(예: Lambda용 Parameters and Secrets Extension) Higher throughput을 켠다.
- **SecureString 메타데이터는 평문**이다. 이름·설명에 민감 정보를 넣지 않는다.
- **`aws/ssm` 기본 키의 한계**: 키 정책을 제어할 수 없고 계정 간 공유도 안 된다. AWS 보안 모범 사례는 기본 키를 쓰면 `GetParameter(s)` 권한만 있는 사용자가 모든 SecureString을 복호화할 수 있다고 지적한다. 접근 범위를 KMS 키 정책으로도 나누려면 customer managed key를 쓴다. ([Security best practices](https://docs.aws.amazon.com/systems-manager/latest/userguide/security-best-practices.html))
- **Advanced는 되돌릴 수 없다**: Intelligent-Tiering이나 실수로 Advanced가 된 파라미터는 다운그레이드가 안 되어 비용이 계속 나간다.
- **정책 실행 시각이 부정확하다**: Expiration은 비동기 스캔이라 지정 시각에 정확히 삭제되지 않을 수 있다.

## 출처

- https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-a-parameter.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-advanced-parameters.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/ps-default-tier.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-policies.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-throughput.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-labels.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-public-parameters-ami.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/integration-ps-secretsmanager.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-shared-parameters.html
- https://docs.aws.amazon.com/kms/latest/developerguide/services-parameter-store.html
- https://docs.aws.amazon.com/general/latest/gr/ssm.html#parameter-store
- https://aws.amazon.com/systems-manager/pricing/
- https://aws.amazon.com/secrets-manager/pricing/
- https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_limits.html

---
*조사 기준일: 2026-10-12. Lambda Parameters and Secrets Extension은 이번 조사에서 1차 출처를 따로 확인하지 않았다(확인 필요).*
