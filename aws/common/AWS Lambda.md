# ✔️ AWS Lambda
`데이터 전처리`
`데이터 필터링/변환`
`이상감지 로직`
>동작 방식, 이벤트 트리거, 기본 구조 (handler, event, context), 연결 대상 예시

https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/concepts-basics.html

<br>

## ✅ Lambda 함수 및 함수 핸들러
### 🔷 Lambda 함수
: 애플리케이션을 생성하는기본 구성 요소
웹 사이트에서 버튼을 클릭하거나, S3 버킷에 파일 업로드 등의 이벤트에 대한 응답으로 실행되는 코드

<br>

### 🔷 Lambda 함수 핸들러
: Lambda가 실행할 함수(메서드)  
: 이벤트 발생 시, Lambda 함수 내부 정의된 핸들러 함수를 실행
```python
def my_handler(event, context):
    # 이 함수가 Lambda의 "핸들러"
    print("Hello Lambda!")
```
Lambda 설정에서 이 함수를 핸들러로 지정한다:

	예: lambda_function.my_handler
🔥 Lambda가 실행될 때는 오직 한 핸들러 함수만 호출된다!!   
▶ Lambda에서 여러 작업을 하고 싶다면, 핸들러 함수 안에서 다른 함수를 호출!

🔥 함수를 실행한 이벤트에 대한 데이터는 핸들러에 직접 전달된다!!

<br>

## ✅ Lambda 실행 환경 및 런타임
### 🔷 실행 환경
Lambda 함수는 Lambda가 관리하는 격리된 실행 환경 내에서 실행된다.  
실행 환경에서는 함수 실행에 필요한 프로세스와 리소스를 관리한다.
- 함수가 처음 호출되면, Lambda는 함수가 실행될 새 실행 환경을 생성한다.
- 함수 실행이 완료된 후에도 Lambda는 실행 환경을 바로 중지하지 않고, 재호출 시, 기존 실행 환경을 재사용한다.

<br>

- `언어별 런타임` (엔진, 라이브러리)
- `핸들러 로직 실행기` (이벤트를 핸들러 함수로 전달하고 결과를 받아 응답으로 처리)
- `/tmp 저장소` (임시 파일 저장 공간)
- `환경 변수` (Lambda Console 환경 값)
- `IAM 역할 기반 권한` (접근 Role)

<br>

### 🔷 런타임
- 실행 환경 내에는 Lambda와 함수 간에 이벤트 정보 및 응답을 릴레이하는 언어별 환경인 런타임도 포함한다.
- 해당 언어로 작성한 핸들러 함수가 실행될 수 있도록 내부에서 이벤트와 응답을 처리하는 구조를 갖는다.

<br>

## ✅ 이벤트 및 트리거
### 🔷 함수 실행 방법
1. `직접 호출`
Lambda Console, AWS CLI, AWS SDK 중 하나를 사용하여 Lambda 함수 직접 호출 가능

2. `간접 호출`
특정 이벤트에 대한 응답으로 다른 AWS 서비스에서 함수를 간적적으로 호출하는 경우가 일반적

<br>

### 🔷 트리거
함수를 이벤트 소스에 연결한다.
함수에는 여러 가지 트리거가 존재할 수 있다.

이벤트가 발생하면 Lambda는 이벤트 데이터를 JSON 문서로 수신한 후 해당 문서를 코드가 처리할 수 있는 객체로 변환한다.   
▶ Lambda 런타임은 JSON을 함수의 핸들러에 전달하기 전에 객체로 변환한다.


<br>

## ✅ 트리거 vs. 이벤트 소스 매핑

### 🔷 Trigger(호출)
이벤트가 발생했을 때, Lambda 함수를 즉시 실행시키는 방식 (API Gateway, S3 업로드 등)
- 이벤트 1건 → Lambda 1회 실행
- 실시간 대량 유입 시 문제점
  - Lambda 과도한 동시 실행
  - 동시 실행 제한 초과
  - 비용 폭증 (과금)
  - API/DB 자원 고갈 (연결제한, 타임아웃)
  - 제어 불가능한 처리 폭주

<br>

#### ➕ Lambda 자동 실행을 위한 트리거 종류
![image](https://github.com/user-attachments/assets/f3aba7ce-b2c1-4aa9-8832-e42c0d2c7ae8)

1. `Lambda를 EventBridge로 자동 실행 (스케줄링)`
```bash
# 5분마다 실행하는 EventBridge rule 생성 (CLI 예시)
aws events put-rule \
  --schedule-expression "rate(5 minutes)" \
  --name "Every5Minutes"

aws lambda add-permission \
  --function-name your-lambda-func \
  --statement-id "AllowExecutionFromEventBridge" \
  --action 'lambda:InvokeFunction' \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:region:account-id:rule/Every5Minutes

aws events put-targets \
  --rule "Every5Minutes" \
  --targets "Id"="1","Arn"="your-lambda-arn"
```

2. `Lambda를 IoT Rule로 자동 실행 (실시간 처리)`
Rule의 Action을 Lambda 함수 호출로 지정하면, 센서가 MQTT 메시지를 보낼 때마다 Lambda 자동 실행!!
```sql
-- IoT SQL Example
SELECT *
FROM 'sensor/topic'
```



<br>

### 🔷 Event Source Mapping(폴링 주기)
Lambda가 SQS, Kinesis 등 대기열/스트림에서 데이터를 감시(폴링)하고, 새 데이터가 있으면 Lambda를 간접 호출하는 방식
- 주기적으로 Queue/Stream 폴링하여 Batch size만큼 이벤트를 가져와 처리
- 대량 유입 시 문제점
  - 지연 발생
  - 배치 처리 실패 시 전체 실패
  - 처리 속도보다 유입 속도가 빠르면 큐 증가
  - 최대 배치 제한 존재
  - 재시도 및 DLQ(Dead Letter Queue) 미설정 시, 데이터 유실 위험

<br>

>### ➕ Amazon Kinesis & Amazon SQS
Amazon Kinesis 또는 Amazon SQS 같은 스트림 대기열 서비스는 표준 트리거 대신 `이벤트 소스 매핑`을 사용한다.
>
SQS, Kinesis는 스트리밍 또는 대기열 기반 서비스이기 때문에 이벤트 발생이 명확하지 않기 때문이다.
>
#### ex) SQS와 Lambda
SQS에 메시지 들어감 > Lambda는 지속적으로 Queue 폴링 > 새 메시지 감지되면, Lambda가 배치로 묶어서 함수에 전달 > Lambda 함수 실행
>
#### ex) Kinesis와 Lambda
실시간 스트리밍 로그를 배치 단위로 수집하여 처리


<br>

## ✅ 실행 오류 종류

### 🔷 타임 아웃
Lambda 기본 실행 시간은 최대 15분  
수천 건의 데이터를 처리하느라 시간이 오래 걸리면 실행 도중 시간 초과  
▶ 배치 크기 제한 + 소스 매핑

### 🔷 메모리 초과
Lambda는 최대 10GB 메모리 제한  
처리중인 데이터가 너무 많아 RAM이 부족할 경우 오류 발생  
▶ 데이터 나눠 처리 + Glue 등으로 이동

### 🔷 API 호출 제한
Lambda 내부에서 외부 API, DB 등을 동시에 많이 호출하면, 연결 제한, rate limit, throttling 발생  
▶ 큐잉 또는 비동기 처리 도입


<br>

## ✅ Lambda 권한 및 역할
🔥 **`최소 권한 원칙`**

### 🔷 함수가 다른 AWS 서비스에 액세스하는 데 필요한 권한 : 실행 역할
IAM 역할의 특수 종류
Lambda가 AWS 리소스 사용하기 위해서는 실행 역할이 반드시 필요하다!
Lambda가 역할을 일시적으로 빌려서 사용한다! (수임) (최소 권한 원칙)


ex) `Lambda가 SQS 메시지를 처리하고 S3에 저장`
- Lambda 함수는 ExecutionRoleForLambda라는 IAM 역할 사용
```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject"],
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```
- Lambda는 SQS 이벤트 소스 매핑에 의해 자동 실행
- 이때 AWS는 Lambda 실행하면서 ExecutionRoleForLambda를 수임 (assume role)
- 이 역할의 권한으로 S3에 데이터 작성 가능

<br>

### 🔷 다른 사용자 및 AWS 서비스가 함수에 액세스하는 데 필요한 권한 : 리소스 기반 정책
다른 AWS 서비스에서 트리거를 통해 함수를 간접적으로 호출하려면 함수의 리소스 기반 정책이 해당 서비스에 

Lambda Console > 함수 페이지 > 함수 선택 > 구성(Configuration) > 권한 (Permissions) > 리소스 기반 정책 > 정책 문서 보기   
▶ `다른 계정 또는 AWS 서비스가 해당 함수에 액세스하려고 할 때 적용되는 권한이 표시되어 있다.`


<br>

## ✅ Lambda 외부 라이브러리 이용 방법
AWS Lambda는 서버리스 함수이다.  
작성한 코드를 실행해주는 환경이지만, 기본 Python 실행 환경만 제공해주고,  
Pandas, Pyarrow, numpy 등 외부 라이브러리들은 제공되지 않는다.

- Lambda는 가벼운 실행 환경이기 때문에 무거운 패키지를 기본 제공하지 않는다.  
▶ 직접 묶거나, 레이어로 따로 관리해야 함!!

<br>

### 🔷 zip 패키징해서 함께 업로드 (local에서 압축)
> 함수 코드 + 외부 라이브러리를 모두 zip으로 묶어서 Lambda에 업로드하는 방법
```
my-function/
├── lambda_function.py
├── pyarrow/
│   └── ...
└── ...
```
1. `pip install pyarrow -t ./my-function/`: 해당 폴더에 라이브러리 파일 다운로드
2. `lambda_function.py`도 같은 폴더에 넣기
3. my-function/ 폴더 자체를 `zip으로 압축`
4. `Lambda에 zip 파일 업로드`

<br>

### 🔷 Lambda Layer 사용
> 외부 라이브러리를 별도로 Layer라는 형태로 등록하고, Lambda 함수에 연결하여 사용하는 방식
```
python/
└── lib/
    └── python3.9/
        └── site-packages/
            └── pyarrow/
```

1. `pip install pyarrow -t python/lib/python3.9/site_packages/` (운영체제 Linux여야 함)
2. python/ 디렉토리를 zip으로 압축 →  `pyarrow-layer.zip`
3. AWS Lambda Console에서 `layer 생성` → `zip 업로드`
4. Lambda 함수에서 해당 `레이어를 추가`


![image](https://github.com/user-attachments/assets/b06c2fbd-64cc-446c-8b44-dcea99b058fa)
