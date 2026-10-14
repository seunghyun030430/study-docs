# 템플릿 문법

> 템플릿의 필수 섹션은 `Resources` 하나다. 값은 **내장 함수**로 조합한다. `!Ref`는 기본값(ID·이름), `!GetAtt`는 속성(ARN 등), `!Sub`는 문자열 조합이다. 참조가 있으면 생성 순서는 CloudFormation이 알아서 정하고, 참조 없이 순서가 필요할 때만 `DependsOn`을 쓴다. 스택 사이에서 값을 넘길 때는 **같은 팀·같은 수명이면 nested stack**, **다른 팀·다른 수명이면 Export/ImportValue**를 쓴다. 비밀번호는 템플릿에 쓰지 않고 **동적 참조**(`{{resolve:secretsmanager:…}}`)로 가져온다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어(템플릿, 스택, 논리 ID, 내장 함수)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 섹션 | 템플릿 최상위 항목(`Parameters`, `Resources`, `Outputs` 등) |
| 의사 파라미터(pseudo parameter) | 따로 정의하지 않아도 쓸 수 있는 값(`AWS::Region`, `AWS::AccountId` 등) |
| 조건(Condition) | 참·거짓에 따라 리소스를 만들지 말지, 값을 무엇으로 할지 정하는 규칙 |
| Mapping | 키로 값을 찾는 고정 표(예: 리전별 AMI ID) |
| Export / ImportValue | 한 스택의 출력값을 같은 계정·리전의 다른 스택에서 가져다 쓰는 방식 |
| Nested stack | 스택 안에 다른 템플릿을 자식 스택으로 넣는 방식 |
| 동적 참조(dynamic reference) | 배포할 때 Parameter Store·Secrets Manager에서 값을 읽어 오는 표기 |
| Transform / 매크로 | 템플릿을 배포 전에 다른 형태로 펼치는 처리(예: SAM) |
| Custom resource | CloudFormation이 지원하지 않는 작업을 Lambda 등으로 직접 구현해 리소스처럼 쓰는 것 |

## 섹션 구성

출처: [Template anatomy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html), [Quotas](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html)

| 섹션 | 필수 | 하는 일 | 한도 |
| --- | --- | --- | --- |
| `AWSTemplateFormatVersion` | 선택 | 현재 유효한 값은 `"2010-09-09"` 하나 | — |
| `Description` | 선택 | 설명 | 1,024바이트 |
| `Metadata` | 선택 | 콘솔 표시 순서 등 부가 정보 | — |
| `Parameters` | 선택 | 배포 시 입력값 | 200개, 값 4,096바이트 |
| `Rules` | 선택 | 파라미터 조합 검증 | — |
| `Mappings` | 선택 | 고정 조회 표 | 200개 |
| `Conditions` | 선택 | 조건 정의 | — |
| `Transform` | 선택 | 매크로 적용(SAM 등) | — |
| **`Resources`** | **필수** | 만들 리소스 | **500개** |
| `Outputs` | 선택 | 출력값, Export | 200개 |

## Parameters

출처: [Parameters](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html), [AWS-specific parameter types](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-supplied-parameter-types.html)

```yaml
Parameters:
  Env:
    Type: String
    AllowedValues: [dev, prod]
    Default: dev
  VpcId:
    Type: AWS::EC2::VPC::Id                       # 계정에 실제로 있는 VPC인지 검사
  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
```

| 타입 | 예 | 메모 |
| --- | --- | --- |
| 기본 | `String`, `Number`, `List<Number>`, `CommaDelimitedList` | `Number`도 `!Ref`로 읽으면 문자열 |
| AWS 전용 | `AWS::EC2::VPC::Id`, `AWS::EC2::KeyPair::KeyName` | 배포할 때 실제로 있는 값인지 검사 |
| SSM 파라미터 | `AWS::SSM::Parameter::Value<String>` | 배포 시점의 Parameter Store 값을 읽음(최신 AMI ID 등, [aws-systems-manager/04](../aws-systems-manager/04-parameter-store.md)) |

- 제약: `AllowedValues`, `AllowedPattern`, `MinLength`/`MaxLength`, `MinValue`/`MaxValue`, `ConstraintDescription`
- `NoEcho: true`는 API 응답에서 값을 `*****`로 가리지만, **`Outputs`와 `Metadata`에 쓰면 그대로 보인다.** 비밀번호는 파라미터보다 동적 참조로 받는다.
- YAML에서 `AllowedValues`에 `Yes`/`No`를 쓰면 불리언으로 해석되므로 따옴표로 감싼다.
- SecureString은 SSM 파라미터 타입으로 쓸 수 없다.

## 내장 함수와 의사 파라미터

출처: [Intrinsic functions](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html), [Pseudo parameters](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html)

| 함수(YAML 짧은 형식) | 하는 일 | 예 |
| --- | --- | --- |
| `!Ref` | 파라미터 값, 또는 리소스의 기본 반환값(리소스마다 다름: 버킷 이름, 인스턴스 ID 등) | `!Ref LogBucket` |
| `!GetAtt` | 리소스의 다른 속성 | `!GetAtt LogBucket.Arn` |
| `!Sub` | 문자열 안에 값 끼워 넣기 | `!Sub "arn:${AWS::Partition}:s3:::${LogBucket}/*"` |
| `!Join` | 목록을 구분자로 합치기 | `!Join [",", [a, b]]` |
| `!Select` / `!Split` | 목록에서 하나 고르기 / 문자열 나누기 | `!Select [0, !GetAZs ""]` |
| `!GetAZs` | 리전의 AZ 목록 | `!GetAZs ""` |
| `!FindInMap` | Mappings에서 값 찾기 | 아래 예시 |
| `!If`, `!Equals`, `!And`, `!Or`, `!Not` | 조건 | 아래 예시 |
| `!ImportValue` | 다른 스택의 Export 값 | 아래 예시 |
| `!Base64` | 문자열을 Base64로(EC2 UserData) | — |
| `!Cidr` | CIDR 블록 나누기 | `!Cidr [!GetAtt Vpc.CidrBlock, 6, 8]` |
| `Fn::GetStackOutput` | 다른 스택 출력값을 **계정·리전을 넘어** 읽기(신규) | 아래 참고 |

| 의사 파라미터 | 값 |
| --- | --- |
| `AWS::Region`, `AWS::AccountId`, `AWS::StackName`, `AWS::StackId` | 배포되는 리전, 계정, 스택 정보 |
| `AWS::Partition`, `AWS::URLSuffix` | `aws`/`aws-cn`/`aws-us-gov`, `amazonaws.com` 등. ARN을 하드코딩하지 않을 때 씀 |
| `AWS::NoValue` | `!If`의 결과로 쓰면 **그 속성을 아예 뺌** |

**짧은 형식 중첩 함정**: `!Base64 !Sub ...`처럼 짧은 형식 함수 안에 짧은 형식 함수를 바로 넣을 수 없다. 둘 중 하나는 긴 형식(`Fn::Base64: !Sub ...`)으로 써야 한다. `!ImportValue`와 `!Sub`도 마찬가지다. ([Fn::Base64](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-base64.html))

### 조건과 Mappings 예시

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Parameters:
  Env:
    Type: String
    AllowedValues: [dev, prod]
Mappings:
  EnvConfig:
    dev:
      RetentionDays: 7
    prod:
      RetentionDays: 365
Conditions:
  IsProd: !Equals [!Ref Env, prod]
Resources:
  AppLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub "/app/${Env}"
      RetentionInDays: !FindInMap [EnvConfig, !Ref Env, RetentionDays]
  AuditBucket:
    Type: AWS::S3::Bucket
    Condition: IsProd                      # prod일 때만 생성
    Properties:
      VersioningConfiguration:
        Status: Enabled
      Tags:
        - Key: env
          Value: !Ref Env
Outputs:
  AuditBucketName:
    Condition: IsProd
    Value: !Ref AuditBucket
```

### 반복: AWS::LanguageExtensions

`Transform: AWS::LanguageExtensions`를 선언하면 `Fn::ForEach`(반복), `Fn::Length`, `Fn::ToJsonString`을 쓸 수 있다. 이 함수들은 YAML 짧은 형식(`!ForEach`)을 지원하지 않고, `Resources`·`Conditions`·`Outputs` 섹션에서만 쓸 수 있다. ([LanguageExtensions](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/transform-aws-languageextensions.html))

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::LanguageExtensions
Resources:
  Fn::ForEach::Queues:
    - QueueName
    - [orders, payments, emails]
    - ${QueueName}Queue:
        Type: AWS::SQS::Queue
        Properties:
          QueueName: !Sub "${AWS::StackName}-${QueueName}"
```

## 의존성

출처: [DependsOn](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-dependson.html)

- `!Ref`, `!GetAtt`, `!Sub`로 다른 리소스를 참조하면 **자동으로 의존성이 생긴다.** 참조되는 쪽이 먼저 만들어지고, 나중에 지워진다.
- 참조가 없으면 **가능한 한 병렬로** 만든다.
- 참조 없이 순서가 필요한 경우에만 `DependsOn`을 쓴다. 대표적인 예가 **인터넷 게이트웨이 연결**이다. 공인 IP가 필요한 EC2·EIP·라우트는 게이트웨이 연결(`VPCGatewayAttachment`)이 끝난 뒤 만들어져야 한다.

```yaml
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: GatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
```

두 리소스가 서로를 참조하면 **순환 의존성(circular dependency)** 오류가 난다(→ [06](./06-issues.md)).

## 스택 사이에서 값 넘기기

출처: [Exports](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html), [Nested stacks](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html), [Fn::GetStackOutput](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-getstackoutput.html)

| 방법 | 범위 | 특징 | 언제 |
| --- | --- | --- | --- |
| **Export / `!ImportValue`** | 같은 계정·리전 | **강한 연결**: 누가 import하는 동안 export하는 스택을 지우거나 그 값을 바꿀 수 없음 | 네트워크 팀의 VPC를 앱 팀이 가져다 씀(수명이 다른 스택) |
| **Nested stack** | 부모·자식 스택 | 한 번에 같이 배포·삭제, `!GetAtt Child.Outputs.X`로 값 전달 | 한 팀이 큰 템플릿을 여러 파일로 나눌 때 |
| **`Fn::GetStackOutput`** | 계정·리전을 넘어 | **약한 연결**: 배포 시점에 읽기만 함, 지우거나 바꿔도 막지 않음 | 다른 계정·리전의 값을 읽을 때 |

```yaml
# network 스택
Outputs:
  VpcId:
    Value: !Ref Vpc
    Export:
      Name: !Sub "${AWS::StackName}-VpcId"

# app 스택
Resources:
  AppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: app
      VpcId: !ImportValue network-prod-VpcId
```

- Export 이름은 계정·리전 안에서 고유해야 한다. `list-exports`와 `list-imports`로 누가 쓰는지 확인한다.
- Export 값을 바꿔야 하는데 import하는 스택이 있으면 업데이트가 실패한다. 먼저 import하는 쪽을 다른 값으로 바꾼다(→ [06](./06-issues.md)).
- nested stack 템플릿은 S3에 있어야 한다. `aws cloudformation package`가 로컬 템플릿을 S3에 올리고 경로를 바꿔 준다. 업데이트는 **최상위(root) 스택에서** 시작한다.
- `Fn::GetStackOutput`은 Sub의 변수 맵 안, Base64 안, Outputs 값 자체 등 일부 위치에서 쓸 수 없다. 언제 출시되었는지는 확인하지 못했다.

## 동적 참조: 비밀 값을 템플릿에 쓰지 않기

출처: [Dynamic references](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html)

| 표기 | 읽는 곳 | 메모 |
| --- | --- | --- |
| `{{resolve:ssm:/app/prod/db/host}}` | Parameter Store(String) | 버전을 생략하면 배포 때마다 최신 값. drift 감지 안 됨 |
| `{{resolve:ssm-secure:/app/prod/db/password:3}}` | Parameter Store(SecureString) | **지원하는 속성이 정해져 있음**(RDS `MasterUserPassword` 등). CloudFormation이 값을 저장하지 않음 |
| `{{resolve:secretsmanager:prod/db:SecretString:password}}` | Secrets Manager | 모든 리소스 속성에서 사용 가능. 기본은 `AWSCURRENT` 버전 |

```yaml
  AppDatabase:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    UpdateReplacePolicy: Snapshot
    Properties:
      Engine: postgres
      DBInstanceClass: db.t4g.micro
      AllocatedStorage: "20"
      MasterUsername: "{{resolve:secretsmanager:prod/db:SecretString:username}}"
      MasterUserPassword: "{{resolve:secretsmanager:prod/db:SecretString:password}}"
```

- 템플릿당 60개까지 쓸 수 있다. EC2 `UserData`와 `AWS::CloudFormation::Init`에서는 쓸 수 없다.
- **Secrets Manager에서 값만 바꿔도 CloudFormation은 다시 읽지 않는다.** 그 참조를 가진 리소스가 수정될 때만 새로 읽는다.
- RDS 비밀번호는 `ManageMasterUserPassword: true`로 RDS가 Secrets Manager에 직접 관리하게 하는 방법도 있다([03](./03-stack-operations.md#deletionpolicy--updatereplacepolicy) 예시).

## Transform과 Custom resource

| 기능 | 하는 일 | 출처 |
| --- | --- | --- |
| `Transform: AWS::Serverless-2016-10-31` | SAM 문법(`AWS::Serverless::Function` 등)을 일반 리소스로 펼침 | [Transform](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/transform-section-structure.html) |
| `AWS::Include` | S3의 템플릿 조각을 끼워 넣음 | 같음 |
| `AWS::LanguageExtensions` | ForEach 등 추가 함수 | 위 참고 |
| **Custom resource**(`Custom::Name`) | Lambda·SNS가 Create/Update/Delete 요청을 받아 처리하고, 결과를 S3 URL로 응답 | [Custom resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html) |

- Transform이나 매크로가 들어간 템플릿을 바로 배포하려면 `CAPABILITY_AUTO_EXPAND`가 필요하다.
- Custom resource가 **응답하지 않으면 기본 1시간(3600초)을 기다린 뒤 실패**한다. `ServiceTimeout`으로 줄일 수 있다. 응답 데이터는 4,096바이트까지다.
- Lambda 코드를 `ZipFile`로 인라인에 넣으면 `cfnresponse` 모듈을 쓸 수 있다. S3에 올린 코드는 응답 전송을 직접 구현해야 한다.

## 이슈 / 주의

- **Outputs는 가려지지 않는다.** 비밀 값을 Output으로 내보내지 않는다.
- **이름을 지정한 리소스는 교체할 수 없다**(→ [03](./03-stack-operations.md#업데이트-동작-수정인가-교체인가)). 꼭 필요할 때만 이름을 지정한다.
- **Export는 강한 연결**이라 나중에 바꾸기 어렵다. 자주 바뀌는 값은 Parameter Store에 넣고 SSM 파라미터 타입으로 읽는 방법도 있다.
- **ARN·리전을 하드코딩하지 않는다.** `AWS::Partition`, `AWS::Region`, `AWS::AccountId`를 쓰면 다른 리전·파티션에서도 그대로 배포된다.
- **Custom resource는 오류 처리를 꼼꼼히** 한다. 예외로 응답을 못 보내면 스택이 1시간 동안 멈춘다.

## 출처

- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-supplied-parameter-types.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-base64.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/transform-aws-languageextensions.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-dependson.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-getstackoutput.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/transform-section-structure.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html

---
*조사 기준일: 2026-10-14. 템플릿 예시는 cfn-lint 1.57.2(`--regions ap-northeast-2`)로 검사해 오류·경고 없이 통과했다. 리소스 하나만 보여 주는 조각은 필요한 리소스를 덧붙인 최소 템플릿으로 검사했다. 실제 계정에 배포하지는 않았다.*
