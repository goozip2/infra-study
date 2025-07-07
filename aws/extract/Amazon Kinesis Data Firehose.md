# ✔️ Amazon Kinesis Data Firehose

https://docs.aws.amazon.com/ko_kr/firehose/latest/dev/what-is-this-service.html

실시간 적재
>S3 등으로 실시간 적재 흐름, 데이터 변환 기능

<br>

## ✅ Kinesis
Kinesis는 여러 서비스 구성 요소로 이루어진 스트리밍 플랫폼이다.

### 🔷 Data Stream
: 고속 스트리밍, 실시간 소비자(Lambda, Flink 등)로 직접 처리

### 🔷 Data Firehose
: 스트림을 S3, Redshift, OpenSearch 등으로 자동 전달하는 관리형 솔루션

### 🔷 Data Analytics
: SQL 기반 실시간 스트림 처리 (Flink 기반)

<br>

## ✅ Kinesis Data Firehose
`버퍼링` `전송` `변환`   

: 스트리밍 데이터를 자동으로 버퍼링(buffer) → 대상 저장소로 배치 전송하는 완전 관리형 서비스  
: 실시간 스트리밍 데이터를 전송하기 위한 완전 관리형 서비스  

데이터 생산자가 Amazon Data Firehose로 데이터를 보내도록 구성하면, 지정한 대상으로 데이터가 자동 전송된다.  
전송 전에 데이터를 변환하도록 구성할 수도 있다.

- 대상 저장소: S3, Redshift, OpenSearch, HTTP Endpoint 등 다양  
- 실패 시, S3 백업 경로(ErrorOutputPrefix) 설정 가능

<br>

## ✅ 실무 흐름 예시
```
[ IoT 센서 / 앱 / 서버 로그 ]
        ↓
[ Kinesis Data Firehose (버퍼링, 변환, 압축) ]
        ↓
[ 저장소 ]
  - S3 (Parquet, JSON)
  - Redshift (COPY via S3)
  - OpenSearch (검색/시각화)
        ↓
[ Glue Crawler → 테이블 자동 생성 ]
        ↓, Redshift
[Glue Data Catalog (메타데이터 DB)]
        ↓
[ Athena, Kibana, Grafana, QuickSight, Redshift 분석 ]
```

>### ➕ Firehose vs. Crawler
>#### 🔸 Kinesis Data Firehose
>: 실시간으로 데이터를 받아 S3에 저장  
>raw 데이터 파일 저장
>
>#### 🔸 Crawler
>: s3에 저장된 파일을 주기적으로 크롤링해서 스키마를 자동 추론  
>Glue Data Catalog에 테이블 생성  
>▶ Crawler는 Firehose의 출력 결과인 S3를 입력으로 사용한다!!

<br>

## ✅ 주요 개념

### 🔷 Firehose 스트림
: Amazon Data Firehose의 기본 Entity  
데이터를 Firehose 스트림으로 전송하여 Amazon Data Firehose를 사용한다.

### 🔷 레코드
: 데이터 생산자가 Firehose 스트림으로 보내는 관심 있는 데이터

### 🔷 데이터 생산자
Firehose 스트림에 레코드를 전송한다.  
Firehose 스트림에 로그 데이터를 보내는 웹 서버가 데이터 생산자이다.

### 🔷 버퍼 크기와 버퍼 간격
Amazon Data Firehose는 수신되는 스트리밍 데이터를 대상으로 전송하기 전에 특정 기간 또는 특정 크기로 버퍼링한다.  
- `Buffer Size` 단위는 `MB`
- `Buffer Interval` 단위는 `sec`


<br>

## ✅ Amazon Data Firehose의 데이터 흐름 이해
### 🔷 Amazon S3 대상
1. 스트리밍 데이터가 S3 버킷으로 전송된다.
2. 데이터 변환 활성화된 경우, 선택적으로 소스 데이터를 다른 Amazon S3 버킷으로 백업한다.

![](https://velog.velcdn.com/images/goozip2/post/0603e59e-45a3-4b85-a909-e2c1d76758c7/image.png)

```yaml
Resources:
  MyFirehose:
    Type: AWS::KinesisFirehose::DeliveryStream   # Firehose 리소스 타입
    Properties:
      DeliveryStreamName: firehose-to-xxx        # 원하는 스트림 이름
      DeliveryStreamType: DirectPut              # 입력 타입 (DirectPut or KinesisStreamAsSource)
      S3DestinationConfiguration:
        RoleARN: arn:aws:iam::123456789012:role/FirehoseS3AccessRole
        BucketARN: arn:aws:s3:::my-s3-destination-bucket
        Prefix: logs/                   # S3 저장 경로 접두사
        CompressionFormat: GZIP        # 압축 형식 (GZIP, Snappy, UNCOMPRESSED 등)
        BufferingHints:                # 배치 전송 조건
          IntervalInSeconds: 300       # 시간 기준 배치 간격
          SizeInMBs: 5                 # 크기 기준 배치 간격
        ErrorOutputPrefix: error/      # 실패 시 저장 경로
```
- 기본형 전송 대상
- Buffering 후 S3에 저장
- Lambda로 변환도 추가 가능 (JSON > Parquet)

<br>

### 🔷 Amazon Redshift 대상
1. 스트리밍 데이터가 먼저 S3 버킷으로 전송한다.
2. Firehose는 Resfhit COPY 명령얼 실행하여 S3 버킷의 데이터를 Redshift 클러스터로 로드한다.  
🔥 Firehose는 S3를 중간 버퍼링 저장소로 두고 데이터를 안정적으로 쌓아둔 뒤, Redshift로 복사하는 안전한 배치 처리 방식을 기본으로 한다!!
3. 데이터 변환이 활성화된 경우, 선택적으로 소스 데이터를 다른 Amazon S3 버킷으로 백업한다.

![](https://velog.velcdn.com/images/goozip2/post/d19f0d60-69b6-47e5-b1fe-8c416b0ec0fa/image.png)

```yaml
Resources:
  MyFirehose:
    Type: AWS::KinesisFirehose::DeliveryStream   # Firehose 리소스 타입
    Properties:
      DeliveryStreamName: firehose-to-xxx        # 원하는 스트림 이름
      DeliveryStreamType: DirectPut              # 입력 타입 (DirectPut or KinesisStreamAsSource)
      RedshiftDestinationConfiguration:
        RoleARN: arn:aws:iam::123456789012:role/FirehoseRedshiftAccessRole
        ClusterJDBCURL: jdbc:redshift://<cluster-url>:5439/dev
        CopyCommand:
          DataTableName: my_table
          CopyOptions: "FORMAT AS JSON 'auto'"
        Username: myuser
        Password: mypassword
        S3Configuration:                 # 임시 S3 저장소 (필수)
          RoleARN: arn:aws:iam::123456789012:role/FirehoseRedshiftAccessRole
          BucketARN: arn:aws:s3:::my-redshift-staging
          Prefix: redshift-data/


```

- Redshift로 바로 보내는 것이 아니라 `S3 → COPY 명령어 → Redshift`
- `S3Configuration` 필수 (중간에 S3 버킷 과정 필수)
  - 대량 데이터를 효율적으로 배치 적재하기 위함
  - S3는 객체 스토리지로, 실시간 대량 데이터 저장과 확장에 유연하고 부담이 적음
  - Redshift는 컬럼형 DW로, 실시간 스트리밍 데이터 적재에 적합하지 않고, 대량 배치 적재에 최적화되어 있음  
▶ 데이터가 실시간으로 Firehose에 들어오면 먼저 S3에 임시 저장하고, S3에 쌓인 데이터를 일정 크기 또는 시간 단위로 묶어서 Redshift에 COPY 명령으로 배치 적재
- 데이터 포맷 명시 필요 (JSON, CSV 등)
- CopyOptions로 압축 형식, 포맷 지정 가능

<br>

>❓ Firehose 자체가 buffer를 사용하는 배치 처리 도구인데, Redshift에 전달하기 위해 S3를 또 거쳐야 하는 이유가 뭘까?  
><br>
>▶ [1] Redshift COPY 명령 특성 때문이다.   
>Redshift는 데이터 적재 시, 대량의 파일을 한꺼번에 COPY 명령으로 읽어들이는 방식을 사용한다.  
><br>
>▶ [2] 안전한 배치 처리를 하기 위함이다.  
>만약 중간 저장소 없이 Redshift에 직접 실시간 쓰기를 시도할 경우,
>- 네트워크 장애, 처리 지연, 오류 발생 시 복구 어려움  
>- 데이터 유실 가능성 증가  
>- Redshift 클러스터 과부하 우려  

<br>

### 🔷 OpenSearch Service 대상
스트리밍 데이터가 OpenSearch Service 클러스터로 전송되며, 동시에 선택적으로 S3 버킷에 백업 가능하다.

![](https://velog.velcdn.com/images/goozip2/post/8833fcb4-9d64-497a-8e0a-9ec7831e275a/image.png)

```yaml
Resources:
  MyFirehose:
    Type: AWS::KinesisFirehose::DeliveryStream   # Firehose 리소스 타입
    Properties:
      DeliveryStreamName: firehose-to-xxx        # 원하는 스트림 이름
      DeliveryStreamType: DirectPut              # 입력 타입 (DirectPut or KinesisStreamAsSource)
      AmazonOpenSearchServiceDestinationConfiguration:
        RoleARN: arn:aws:iam::123456789012:role/FirehoseOSAccessRole
        DomainARN: arn:aws:es:region:123456789012:domain/my-domain
        IndexName: iot-logs
        TypeName: _doc                      # 기본 타입
        IndexRotationPeriod: OneDay        # 인덱스 회전 주기 (None, OneHour, OneDay 등)
        BufferingHints:
          IntervalInSeconds: 300
          SizeInMBs: 5
        RetryOptions:
          DurationInSeconds: 60
        S3BackupMode: AllDocuments
        S3Configuration:
          RoleARN: arn:aws:iam::123456789012:role/FirehoseOSAccessRole
          BucketARN: arn:aws:s3:::os-backup-bucket
          Prefix: backup/
```
- 실시간 데이터 분석에 유리 (대시보드 시각화)
- 인덱스 기반 검색/필터 지원
- 장애 시 데이터를 백업하기 위해 S3 백업이 거의 필수
- Kibana와 시각화 용도로 자주 연계된다.

<br>

### ➕ Lambda 변환 예시
#### 1. Lambda 함수 생성용 CloudFormation YAML  
JSON to Parquet   
🔥 pyarrow 모듈이 필요하므로, 일반적으로는 별도로 패키징한 zip 파일로 배포하거나 Lambda Layer를 사용해야함!!
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Lambda for transforming JSON to Parquet for Firehose

Resources:
  JsonToParquetFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: json-to-parquet-transform
      Runtime: python3.9
      Role: arn:aws:iam::123456789012:role/LambdaExecutionRole
      Handler: lambda_function.lambda_handler
      Timeout: 60
      MemorySize: 512
      Code:
        ZipFile: |
          import base64
          import json
          import pyarrow as pa
          import pyarrow.parquet as pq
          import io

          def lambda_handler(event, context):
              output = []
              for record in event['records']:
                  payload = base64.b64decode(record['data'])
                  data = json.loads(payload)

                  table = pa.Table.from_pydict({k: [v] for k, v in data.items()})
                  buffer = io.BytesIO()
                  pq.write_table(table, buffer)

                  output_record = {
                      'recordId': record['recordId'],
                      'result': 'Ok',
                      'data': base64.b64encode(buffer.getvalue()).decode('utf-8')
                  }
                  output.append(output_record)

              return {'records': output}

```


<br>

#### 2. Firehose DeliveryStream YAML (Lambda 처리 포함)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Firehose with Lambda processing (JSON → Parquet)

Resources:
  MyFirehoseToS3:
    Type: AWS::KinesisFirehose::DeliveryStream
    Properties:
      DeliveryStreamName: firehose-json-to-parquet
      DeliveryStreamType: DirectPut
      S3DestinationConfiguration:
        BucketARN: arn:aws:s3:::my-iot-parquet-bucket
        RoleARN: arn:aws:iam::123456789012:role/FirehoseS3AccessRole
        Prefix: transformed/
        CompressionFormat: UNCOMPRESSED
        BufferingHints:
          IntervalInSeconds: 300
          SizeInMBs: 5
        # Lambda 처리 포함
        ProcessingConfiguration:
          Enabled: true
          Processors:
            - Type: Lambda
              Parameters:
                - ParameterName: LambdaArn
                  ParameterValue: !GetAtt JsonToParquetFunction.Arn

```

1) JsonToParquetFunction Lambda가 JSON → Parquet 변환 수행
2) Firehose가 이 Lambda를 처리기로 설정
3) Firehose는 수집한 데이터를 Lambda로 전달하여 변환
4) 결과 Parquet 데이터를 S3에 저장



<br>


## ✅ Firehose 실무 주요 기능
- `버퍼링 기준 설정`: 최소/최대 버퍼 사이즈
- `Lambda 변환 옵션`: 배포 전 실시간 데이터 변환 가능
- `파일 포맷 변환`: JSON → Parquet, ORC 등으로 자동 변환
- `압축 설정 가능`: gzip, snappy 압축 지원
- `대상 지정`: S3, Redshift, OpenSearch, HTTP 등
- `IAM Role`: Firehose가 대상 리소스에 쓰기 권한 가짐
