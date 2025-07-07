# ✔️ AWS IoT Core
https://docs.aws.amazon.com/ko_kr/iot/latest/developerguide/iot-gs-first-thing.html

>데이터 수집

>MQTT 프로토콜 이해, 인증서/Thing 개념, 메시지 흐름, Rule Action (→ Lambda, S3 등)

<br>

## ✅ 디바이스 선택
- Amazon EC2
- Windows, Linux, Mac
- Raspberry Pi

<br>

## ✅ 디바이스 설정 (PC)
### 🔷 1. Git, Python 설치
### 🔷 2. AWS IoT Device SDK for Python 설치
`AWS IoT Core에 MQTT 연결/통신을 위한 코드 작성 준비`

- SDK (Software Development Kit: 라이브러리+도구)
- Thing 등록은 AWS Console, CLI, SDK 어디서든 가능하지만
- 디바이스에서 AWS IoT Core로 실제 연결(MQTT 통신 등)을 하기 위해서는 SDK 필요
  - AWS IoT Device SDK 사용해서 코드로 통신 수행
  
### 🔷 3. IoT Console에서 Thing 생성 및 인증서 발급
`Console Thing 목록에 MyThing 등록`
- AWS IoT Console 모든 디바이스 > 사물 > 사물 생성
- 인증서/프라이빗 키/Root CA 자동 발급
- 인증서에 정책 연결
- 인증서 및 키 다운로드

### 🔷 4. Thing(Local)에 인증서/키 저장 (certs 디렉토리 생성)
`발급받은 인증서 .crt, 키 .key, Root CA .pem 파일을 로컬 디렉토리에 저장`
- Thing을 생성하고 등록할 때 저장한 프라이빗 키, 디바이스 인증서 및 루트 CA 인증서 파일을 certs 디렉토리에 복사
- 대상 디렉토리 각 파일 이름은 테이블 파일 이름과 일치해야 함

![](https://velog.velcdn.com/images/goozip2/post/5df4e3f1-6ea5-45cd-897b-afce063b7f49/image.png)

<br>

## ✅ 정책 설정 (IoT Console)

### 🔷 5. IoT Console에서 정책 생성 및 인증서에 Attach
`정책 생성 (iot:Connect, iot:Publish, iot:Subscribe 등 포함)`
`정책을 인증서에 연결 (Attach)`

사물 정책이 샘플 스크립트에 연결, 구독, 게시 및 수신할 수 있는 권한을 제공하는지 확인해야 한다.

**IoT Console > Thing > Certificates > Policies**

1. AWS IoT Console에 Thing 목록에서 사물 리소스 찾기
2. 사물 리소스 세부 정보 페이지의 인증서 탭에서 사물 리소스에 연결된 인증서 선택하기
3. 인증서 세부 정보 페이지의 정책 탭에서 인증서에 연결된 정책 선택
4. 정책 개요 페이지에서 JSON 편집기를 찾고 정책 문서 편집을 선택하여 필요에 따라 정책 문서를 검토하고 편집한다.
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iot:Publish",
                "iot:Receive"
            ],
            "Resource": [
                "arn:aws:iot:region:account:topic/test/topic"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iot:Subscribe"
            ],
            "Resource": [
                "arn:aws:iot:region:account:topicfilter/test/topic"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iot:Connect"
            ],
            "Resource": [
                "arn:aws:iot:region:account:client/test-*"
            ]
        }
    ]
}
```

<br>


## ✅ MQTT 통신

### 🔷 6. Thing이 MQTT 라이브러리 사용하여 MQTT 메시지 게시 및 구독
>AWS IoT Device SDK for Python의 aws-iot-device-sdk-python-v2/samples디렉터리에 있는 pubsub.py 샘플 스크립트를 실행한다.
>
>https://github.com/aws/aws-iot-device-sdk-python-v2/blob/main/samples/pubsub.py
>
이 스크립트는 디바이스가 MQTT 라이브러리를 사용하여 MQTT 메시지를 게시 및 구독하는 방법을 보여준다.


```bash
python3 pubsub.py --endpoint your-iot-endpoint --ca_file ~/certs/Amazon-root-CA-1.pem --cert ~/certs/device.pem.crt --key ~/certs/private.pem.key
```
- 계정의 AWS IoT Core에 연결
- 메시지 주체 test/topic을 구독하고 해당 주제에 대해 수신하는 메시지 표시
- test/topic 주제에 10개의 메시지 게시


<br>

## ✅ AWS IoT Console에서 메시지 확인
`실시간 메시지 관찰`
`중앙 로그 뷰어`

Thing들이 주고받는 MQTT 메시지를 실시간 관찰 가능
MQTT 통신 흐름을 디버깅하거나, 연결 여부 및 메시지 포맷을 검증할 때 유용

### 🔷 AWS IoT Console > MQTT 테스트 클라이언트
- Subscribe to a topic(주제 구독)에서 test/topic 주제 구독한다.
- 명령줄 창에서 샘플 앱을 재실행하고 AWS IoT Console의 MQTT 클라이언트에서 메시지 확인!!

<br>

## ✅ 공유 구독
: 여러 클라이언트가 한 주제에 대해 구독을 공유할 수 있으며, 무작위 배포를 통해 해당 주제에 게시된 메시지를 한 클라이언트만 수신 가능
- `로드 밸런싱`
- `복원력 향상`
  - 백엔드 애플리케이션의 연결이 끊어질 경우, 브로커가 그룹 내 나머지 구독자에게 로드를 분산
```
# 비공유 구독
sports/tennis
sports/#

# 공유 구독
$share/consumer/sports/tennis
$share/consumer/sports/#
```

<br>

## ✅ IoT Rule 설정
https://docs.aws.amazon.com/ko_kr/iot/latest/developerguide/iot-create-rule.html

1. Rule SQL 작성
: 필터링할 메시지 정의  
SELECT * FROM 'topic/#'

2. Rule Action 지정
: 어떤 작업을 할지 설정  
S3 저장 / Lambda 호출 / SNS 발송 등

4. IAM Role 연결  
: 해당 액션을 수행할 수 있는 권한 부여

<br>

>### 🔷 EX1: temperature 30도 이상일 경우 S3 저장
**📍 Step 1: SQL 작성**
```sql
SELECT temperature, humidity
FROM 'sensors/room1'
WHERE temperature > 30
```
>
**📍 Step 2: Action 설정**
Action 유형: S3
Bucket 이름: iot-temperature-logs
키(prefix): room1/${timestamp()}.json
메시지가 조건에 부합할 경우 S3에 JSON 형식으로 저장
>
**📍 Step 3: IAM Role**
s3:PutObject 권한이 있는 IAM 역할 연결 필요

<br>

>### 🔷 EX2: 모든 메시지를 Lambda 함수로 전달
📍 Rule SQL
```sql
SELECT * FROM 'devices/+/data'
```
>
📍 Action 설정
Lambda 호출
예: processSensorData() 함수
Lambda에서 전달받는 이벤트는 아래처럼 생김:
```json
{
  "temperature": 28.2,
  "humidity": 65,
  "deviceId": "device001"
}
```
