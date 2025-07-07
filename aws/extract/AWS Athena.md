# ✔️ AWS Athena
분석용 쿼리
>SQL 쿼리 방식, Glue 테이블 연계 구조

https://docs.aws.amazon.com/ko_kr/athena/latest/ug/what-is.html

https://docs.aws.amazon.com/ko_kr/athena/latest/ug/notebooks-spark-getting-started.html

표준 SQL을 사용하여 S3에 있는 데이터를 직접 간편하게 분석할 수 있는 대화형 쿼리 서비스

Athena를 사용하면 리소스를 계획, 구성 또는 관리할 필요 없이 Apache Spark를 사용하여 데이터 분석을 대화식으로 쉽게 실행할 수 있다. 

Athena에서 Apache Spark 애플리케이션을 실행하는 경우 처리를 위해 Spark 코드를 제출하고 결과를 직접 수신한다.

Athena Console의 간소화된 노트북 환경을 활용하면 Python 또는 Athena 노트북 API를 통해 Apache Spark 애플리케이션 개발이 가능하다.


<br>


## ✅ Athena Console에서 사용 흐름 (간단 테스트)
AWS Console에서 바로 SQL문을 작성하고 실행할 수 있는 서버리스 쿼리 서비스

1. S3에 데이터 저장 (JSON, Parquet 등)
2. Glue Crawler로 스키마 추출 → Data Catalog에 테이블 생성
3. Athena 콘솔 접속 → 테이블 선택
4. SQL 작성 후 쿼리 실행
5. 결과 확인 (기본적으로 S3에 쿼리 결과 저장)

```sql
SELECT device_id, AVG(temperature)
FROM iot_db.iot_table
WHERE timestamp BETWEEN date '2025-07-01' AND date '2025-07-07'
GROUP BY device_id;
```

<br>

## ✅ Athena IaC 사용 흐름
### 🔷 Athena WorkGroup 정의
: Athena에서 쿼리 실행 단위
```yaml
Resources:
  MyAthenaWorkGroup:
    Type: AWS::Athena::WorkGroup
    Properties:
      Name: "iot-analysis-group"
      Description: "Workgroup for IoT analysis queries"
      WorkGroupConfiguration:
        ResultConfiguration:
          OutputLocation: "s3://my-athena-query-results-bucket/"
        EnforceWorkGroupConfiguration: true
```
- 쿼리 결과 저장 위치, 쿼리 실행 제한 등 별도 설정 가능



### 🔷 결과 저장 S3 버킷 생성
: 쿼리 실행 결과(CSV, 메타 데이터 등)가 자동 저장됨


### 🔷 자주 쓰는 SQL을 NamedQuery로 등록 (선택)
```yaml
Resources:
  AvgTempQuery:
    Type: AWS::Athena::NamedQuery
    Properties:
      Name: "AverageTemperatureQuery"
      Database: "iot_db"
      Description: "Get average temperature by device"
      QueryString: |
        SELECT device_id, AVG(temperature)
        FROM iot_table
        GROUP BY device_id
      WorkGroup: "iot-analysis-group"
```
▶ Athena 콘솔에서 "AverageTemperatureQuery"라는 이름의 SQL이 저장된다.

```
[사용자/애플리케이션]
        ↓ 쿼리 실행
     [Athena 서비스]
        ↓ WorkGroup 지정
[WorkGroup 설정] → [S3 쿼리 결과 저장 위치]
        ↓ 쿼리 실행 후 결과 저장
        ↓ 결과 반환
```


<br>

## ✅ 쿼리 실행 방법
### 🔷 분석가나 개발자가 직접 콘솔에서 쿼리 실행
### 🔷 자동화 스크립트 또는 애플리케이션에서 CLI/SDK 이용한 쿼리 실행
### 🔷 AWS Lambda 등 서버리스 함수와 이벤트 기반 자동 실행
