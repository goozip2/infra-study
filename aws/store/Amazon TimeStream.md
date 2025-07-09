# ✔️ Amazon TimeStream
시계열 DB

> 시계열 DB 개념, 테이블 구조 (dimensions, measures), 기본 쿼리 형태

https://aws.amazon.com/ko/timestream/

: 지연 시간이 짧은 쿼리부터 대규모 데이터 모으기까지 다양한 워크로드에 적합한 완전 관리형 목적별 시계열 데이터베이스 엔진 제공  

완전 관리형이기 때문에 설치, 업그레이드, 저장, 고가용성을 위한 복제, 수동 백업과 같이 시간이 많이 걸리는 DB 인프라 작업을 생략해준다.  

시계열 분석 기능 내장으로, 추세와 패턴을 거의 실시간으로 식별 가능하다.  
TimeStream for InfluxDB를 사용하면 밀리초의 응답 시간으로 실시간 경고 및 인프라 신뢰성 모니터링을 쉽게 실행할 수 있다.  

<br>

## ✅ Timestream 사용 이유
시계열 데이터를 위한 관리형 DB
- `IoT 센서`: 온도, 습도, 위치, 상태 등 시간이 중요한 값
- `서버 모니터링`: CPU 사용률, 메모리, 네트워크 트래픽 등
- `사용자 행동`: 클릭 수, 체류 시간 등 시간 흐름 기반 데이터
- `로깅`: 시간순으로 기록되는 이벤트 데이터

<br>

## ✅ Timestream 구성 요소
- `Databsae`: 시계열 데이터를 저장할 논리 공간
- `Table`: 특정 `센서`나 `서비스` 단위로 데이터 저장
- `Measures`: 실제 측정값 (온도, 습도, 속도 등)
- `Timestamp(Time)`: 언제 측정됐는가 (시계열 핵심)

<br>

## ✅ Lambda 이용 실시간 처리 실습
```
[IoT Device]
     ↓ MQTT (topic: sensor/data)
[AWS IoT Core]
     ↓ IoT Rule (sensor/data 감지)
[Lambda 함수]
     ↓ boto3 SDK 사용 → write_records()
[Timestream DB]
```

<br>

### 🔷 Database / Table 생성
```python
import boto3
timestream = boto3.client('timestream-write')
timestream.create_database(DatabaseName="iot_db")
timestream.create_table(DatabaseName="iot_db", TableName="sensor_data")
```

<br>

### 🔷 Lambda 함수 생성
```python
import boto3
import time
import json

timestream = boto3.client('timestream-write')

DB_NAME = 'iot_db'
TABLE_NAME = 'sensor_data'

def lambda_handler(event, context):
    # IoT Core에서 받은 MQTT 메시지 파싱
    print("Event:", event)
    payload = event['payload']
    record = json.loads(payload)

    device_id = record.get('device_id', 'unknown')
    temperature = str(record.get('temperature', '0'))
    timestamp = str(int(time.time() * 1000))  # 밀리초 기준 현재 시간

    response = timestream.write_records(
        DatabaseName=DB_NAME,
        TableName=TABLE_NAME,
        Records=[
            {
                'Dimensions': [
                    {'Name': 'device_id', 'Value': device_id}
                ],
                'MeasureName': 'temperature',
                'MeasureValue': temperature,
                'MeasureValueType': 'DOUBLE',
                'Time': timestamp,
                'TimeUnit': 'MILLISECONDS'
            }
        ]
    )

    return {
        'statusCode': 200,
        'body': 'Data written to Timestream'
    }
```

<br>

### 🔷 Lambda 권한 부여 (IAM Role)

Lambda 실행 역할에 다음 정책을 추가 필수!

- AmazonTimestreamFullAccess
- AWSIoTDataAccess

<br>

### 🔷 IoT Rule 생성 (Console, CLI)
+) IoT Rule: SQL, Action, Condition, IAM Role
```sql
SELECT *
FROM 'sensor/data'
```
Lambda 함수 실행 → 위에서 만든 Lambda 함수 선택

▶ 콘솔에서 IoT Rule 생성 시 바로 Lambda 연결 가능  
▶ IoT Core가 Lambda를 호출할 수 있도록 권한도 자동 부여됨

<br>

### 🔷 IoT 테스트 (MQTT 메시지 전송)
AWS 콘솔 → IoT Core → MQTT 테스트 클라이언트
```json
{
  "device_id": "sensor-A1",
  "temperature": 28.4
}
```
▶ 메시지 전송 시 Lambda가 실행되고,  
▶ TimeStream에 데이터가 적재됨

<br>

### 🔷 Timestream 쿼리 확인

```python
import boto3

query_client = boto3.client('timestream-query')

query = """
SELECT time, device_id, measure_value::double 
FROM "iot_db"."sensor_data" 
ORDER BY time DESC
LIMIT 5
"""

result = query_client.query(QueryString=query)
for row in result['Rows']:
    print(row)
```

<br>



## ✅ 스케줄링 + Lambda 이용 배치 처리
수집된 로그나 센서 데이터를 S3 같은 저장소에 모았다가,  
주기적으로 AWS Lambda 또는 AWS Glue Job, EC2 작업 등이 실행되어 한꺼번에 TimeStream에 데이터를 쓰거나, TimeStream 내 쿼리를 정기적으로 실행하여 집계, 리포팅 작업을 수행한다.

### 🔷 전체 흐름
```
[IoT Devices] --MQTT--> [AWS IoT Core]
                       ↓
            [IoT Rule → S3 저장 (JSON 파일)]
                       ↓
            [매시간 EventBridge → Lambda 트리거]
                       ↓
          [Lambda: S3에서 JSON 읽고, Timestream에 배치 적재]
                       ↓
              [Timestream에 데이터 저장 완료]

```

<br>

### 🔷 Lambda 예시 코드
```python
import boto3
import json
import time

s3 = boto3.client('s3')
timestream = boto3.client('timestream-write')

DB_NAME = 'iot_db'
TABLE_NAME = 'sensor_data'

def lambda_handler(event, context):
    # S3 이벤트 기반이면 event에서 버킷과 키 받음
    # 스케줄링용이면 버킷+키 직접 지정 가능

    bucket = 'your-bucket-name'
    prefix = 'iot-data/'

    # S3에서 최근 파일 리스트 조회 (필요하면)
    response = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
    if 'Contents' not in response:
        print("No files found")
        return

    for obj in response['Contents']:
        key = obj['Key']
        data_obj = s3.get_object(Bucket=bucket, Key=key)
        data = data_obj['Body'].read().decode('utf-8')
        records = json.loads(data)  # JSON 배열 가정

        timestream_records = []
        for rec in records:
            device_id = rec.get('device_id', 'unknown')
            temperature = str(rec.get('temperature', '0'))
            timestamp = str(int(time.time() * 1000))

            timestream_records.append({
                'Dimensions': [{'Name': 'device_id', 'Value': device_id}],
                'MeasureName': 'temperature',
                'MeasureValue': temperature,
                'MeasureValueType': 'DOUBLE',
                'Time': timestamp,
                'TimeUnit': 'MILLISECONDS'
            })

        # 배치로 쓰기 (최대 100개 권장)
        for i in range(0, len(timestream_records), 100):
            batch = timestream_records[i:i+100]
            try:
                timestream.write_records(
                    DatabaseName=DB_NAME,
                    TableName=TABLE_NAME,
                    Records=batch
                )
                print(f"Wrote {len(batch)} records")
            except Exception as e:
                print("Error writing batch:", e)

    return {'statusCode': 200, 'body': 'Batch write complete'}

```

