# ✔️ AWS S3
>객체 저장 개념, JSON 저장, 버킷 정책 개념

: 파일(객체)를 저장하고 관리할 수 있는 클라우드 스토리지 서비스

https://aws.amazon.com/ko/s3/storage-classes/?nc=sn&loc=3


<br>

## ✅ 기본 개념
- `Bucket`: S3에서 데이터를 저장하는 최상위 컨테이너 (폴더 개념)
- `Object`: S3에 저장된 파일 하나하나 (파일 + 메타데이터)
- `Key`: S3에서 파일의 경로 (고유 경로)
- `Prefix`: S3에서 폴더처럼 동작하는 문자열 (접두어)

<br>

## ✅ 특징
- `무제한 저장 가능`: 이론적으로 용량 제한 없다.
- `객체 기반 저장`: 각 파일은 Object로 저장된다. (메타데이터 포함)
- `버킷 단위로 구성`: 파일은 특정 버킷에 저장됨
- `버전 관리, 암호화, 권한 관리 기능`: 객체별 버전 관리, 퍼블릭 여부, 암호화, 권한 부여 가능

<br>

## ✅ S3 종류
- `S3 Standard`: 자주 읽고 쓰는 개발 초기 단계에 적합
- `S3 Intelligent - Tiering`: AWS가 자동으로 자주/드물게 접근 판단하여 비용 최적화
- `Standard-IA`: 드물게 접근하는 데이터에만 적합
- `Glacier`: 개발 중 분석 속도 느림 (비추천)

<br>

![](https://velog.velcdn.com/images/goozip2/post/9d03ed96-dcc6-4f87-9fcc-1eef91dd6a63/image.png)

<br>

![](https://velog.velcdn.com/images/goozip2/post/8a82bb6f-ab20-46a2-a42e-0d96ef189325/image.png)

### 🔷 예시 상황별 적용

1. IoT 센서 로그 수집 (실시간 모니터링 포함)
→ `S3 Standard`

2. 분석이 자주 되지는 않지만 일단 저장 필요
→ `S3 Intelligent-Tiering`

3. 6개월 지난 데이터는 거의 접근하지 않음
→ `S3 Lifecycle Rule로 Standard-IA` 또는 `Glacier`로 전환 설정


<br>


## ✅ IoT Core의 Rule 엔진에서 적재
```
[IoT Device] → MQTT (sensor/+/data)
   ↓
[IoT Core Rule] → S3 (iot-data-bucket-123)
```
### 🔷 MQTT topic sensor/+/data에 데이터가 오면 해당 JSON을 S3에 저장

```yaml

AWSTemplateFormatVersion: '2010-09-09'
Description: IoT Core to S3 pipeline

Resources:

  # 1. S3 Bucket for IoT Data Storage
  IoTDataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: iot-data-bucket-123  # globally unique

  # 2. IAM Role for IoT to access S3
  IoTRuleRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: iot-s3-access-role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: iot.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: IoTToS3Policy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:PutObject
                Resource: !Sub arn:aws:s3:::${IoTDataBucket}/*
  
  # 3. IoT Topic Rule
  IoTToS3Rule:
    Type: AWS::IoT::TopicRule
    Properties:
      RuleName: IoTToS3Rule
      TopicRulePayload:
        Sql: "SELECT * FROM 'sensor/+/data'"
        RuleDisabled: false
        Actions:
          - S3:
              BucketName: !Ref IoTDataBucket
              Key: "rawdata/${timestamp()}.json"
              RoleArn: !GetAtt IoTRuleRole.Arn

Outputs:
  S3Bucket:
    Description: "IoT S3 Data Bucket"
    Value: !Ref IoTDataBucket
  IoTRole:
    Description: "IAM Role for IoT Rule to access S3"
    Value: !Ref IoTRuleRole


```


<br>

## ✅ 활용 예시

### 🔷 1. 저장된 JSON → Parquet으로 변환 (Firehose 활용)
```
[디바이스 → IoT Core → IoT Rule]
      ↓
[Firehose Delivery Stream (JSON 수신)]
      ↓ (자동 변환)
[S3에 Parquet 포맷으로 저장]
```

- Input Format: JSON (IoT Core Rule로 전달)
- Output Format: Parquet (압축 등 가능)
- Conversion enabled: YES
- Schema 등록: Glue Data Catalog에서 테이블 미리 정의


```json
"DataFormatConversionConfiguration": {
  "Enabled": true,   # 데이터 포맷 변환 기능 활성화 (JSON > Parquet)
  "InputFormatConfiguration": {     # JSON 역직렬화 (테이블 형태로 해석할 수 있도록)
    "Deserializer": {
      "OpenXJsonSerDe": {}
    }
  },
  "OutputFormatConfiguration": {
    "Serializer": {
      "ParquetSerDe": {  # 변환 후 저장할 포맷 지정 (Parquet)
        "Compression": "SNAPPY"   # Parquet의 압축 알고리즘
      }
    }
  },
# Glue Data Catalog에 등록된 테이블 구조를 Firehose가 참조해서 Parquet 변환을 수행
  "SchemaConfiguration": {
    "RoleARN": "arn:aws:iam::1234567890:role/firehose-glue-role",
    "DatabaseName": "iot_data",
    "TableName": "sensor_parquet_table"  # 	데이터 구조가 미리 정의되어 있는 테이블 이름
  }
}
```

🔥 Glue Crawler는 JSON 데이터를 기반으로 스키마를 자동 예측할 수 있지만  
(JSON을 읽고, 필드명·데이터타입 등을 자동으로 추론해 스키마 생성 가능)  
🔥 Kinesis Firehose의 Parquet 변환 기능에서는 `미리 정의된 Glue Table`이 꼭 필요함!!!  
🔥 Firehose는 실시간 처리여서 데이터 흐름 전에 스키마를 알아야 함!!  
(데이터가 흐르기 전에 미리 스키마를 알아야 하기 때문에, Glue 테이블을 미리 정의해야 함)  


<br>

### 🔷 2. S3에 저장된 파일을 Glue Crawler로 분석
```
[S3 (Parquet 저장)]
   ↓
[Glue Crawler 실행 (주기적)]
   ↓
[Glue Data Catalog 테이블 생성]
   ↓
[Athena/Redshift Spectrum 분석]
```


- S3 대상
- Format 자동 인식: Parquet, JSON, CSV 등 자동 처리
- 주기 설정: 매일/매시간 자동 실행 가능
- 결과 저장 위치: Glue Data Catalog의 Table
- Athena 쿼리
```
SELECT device_id, temperature
FROM iot_data.sensor_parquet_table
WHERE timestamp > current_date - interval '1' day;
```

<br>

### 🔷 3. S3 데이터 구조 자동 정렬 (Prefix 분리)

#### 🔸 S3 Prefix란?
S3는 폴더 개념이 없고, 내부적으로는 객체의 Key로 경로를 흉내내는 방식을 사용한다.
```
s3://iot-data/processed/year=2025/month=07/day=08/device_id=sensor-A1/data-12345.json
```
- year=2025/month=07/...은 Key의 일부
- 실제로는 그냥 하나의 긴 문자열

<br>

#### 🔸 이런 구조를 사용하는 이유?
- Athena, Glue, Redshift Spectrum 같은 분석 도구는 `year=2025/month=07/...` 같은 경로 구조를 보고, partition으로 인식한다.

파티션은 데이터를 카테고리별로 나눠서 빠르게 찾도록 해주는 구조다.
전체 파일을 보지 않고, 해당 경로만 탐색한다!

<br>

#### 🔸 중요한 이유?
- Glue/Athena는 partitioned prefix 구조를 기반으로 효율적인 필터링 수행
- 비용 절감, 속도 향상, 정리된 저장 방식 제공
- 자동으로 파티션 컬럼으로 추론하여 테이블을 생성한다.

<br>

#### 🔸 Prefix 생성 방법?
- IoT Rule → S3 Action
IoT Rule에서 S3로 저장할 때 Key(파일 경로)를 직접 정의 가능
```bash
iot-data/processed/year=${timestamp:YYYY}/month=${timestamp:MM}/day=${timestamp:dd}/device_id=${device_id}/data-${newuuid()}.json
```
- Firehose → S3
Firehose도 S3에 저장할 때 prefix 지정 가능
```json
"Prefix": "processed/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/device_id=!{partitionKeyFromQuery:data.device_id}/"

```
