# ✔️ AWS Glue Crawler
스키마 생성
S3 등의 데이터를 스캔하여 자동으로 테이블 생성
>Crawler → 테이블 생성 흐름, S3에 저장된 데이터 자동 인식 원리

+) 실무 흐름: S3에 저장된 JSON → Crawler 실행 → Glue 메타데이터 → Athena에서 SQL 분석

<br>

## ✅ Crawler 기본 원리
AWS Glue Crawler는 S3에 저장된 JSON, CSV, Parquet, ORC 등 다양한 구조화/반구조화된 파일 포맷을 스캔해서 AWS Glue Data Catalog 테이블을 자동 생성해준다.  

Crawler는 모든 필드를 자동 분석하여 테이블화하기 때문에 원하는 항목만 선택하여 스키마를 만들 수 없다.  
▶ Glue Console 또는 Athena에서 테이블 수정 가능  
▶ Glue Job(Python Shell, Spark) 사용 (새 테이블로 가공)  
▶ Athena View 생성 (원하는 필드만 포함하는 가상 테이블 View 생성 가능)

### 🔷 S3 Data 형태
```json
{
  "device_id": "sensor001",
  "temperature": 23.5,
  "timestamp": "2025-07-07T09:30:00Z"
}
```

<br>

### 🔷 Catalog 테이블 형태
🔥 첫 데이터가 중요하다! 첫 데이터 타입으로 설정됨!  
S3 경로 지정 > 각 파일의 구조를 샘플링(1~5개 파일) > 데이터 타입과 필드 추론 > 추론된 스키마로 Glue Data Catalog에 테이블 등록  
![](https://velog.velcdn.com/images/goozip2/post/ca7ec8a2-7590-4399-a21f-9a60d99fe950/image.png)

<br>

## ✅ Parquet vs. JSON
![](https://velog.velcdn.com/images/goozip2/post/1d24ac57-210f-4145-8500-a5a9e44048f2/image.png)
### 🔷 Parquet 장점
1. `칼럼 지향 저장`
필요한 칼럼만 읽어올 수 있다.
ex. 100개 칼럼 중 3개 필요한 경우, parquet는 3개만 읽고, JSON은 전체 파일 파싱 필요

2. `고속 분석에 유리`
Athena, Redshift, Spectrum, Sparkkk 등은 Parquet 형식을 훨씬 빠르고 저렴하게 읽는다.

3. `Glue Crawler 및 Data Catalog와 호환성 좋음`
Parquet 파일은 스키마 구조가 명확하여 Glue가 스키마 추론을 더 빠르고 정확하게 한다.

<br>


## ✅ AWS CloudFormation으로 Glue Crawler 생성하기
+) CloudFormation은 AWS 공식 IAC 도구. YAML, JSON 형식으로 AWS 리소스 선언

### 🔷 YAML 파일 작성 (AWS 리소스 선언)
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Glue Crawler 생성 예제

Resources:
  MyGlueCrawler:
    Type: AWS::Glue::Crawler
    Properties:
      # 크롤러 이름
      Name: my-glue-crawler
      # Glue가 접근할 권한을 가진 IAM Role ARN
      # (S3에 접근 가능한 정책과 Glue 기본 권한을 포함)
      Role: arn:aws:iam::123456789012:role/AWSGlueServiceRole
      # Glue DB명
      DatabaseName: my-glue-db
      # 테이블명 접두사
      TablePrefix: iot_
      # 크롤링 대상 (S3 버킷 경로 등)
      Targets:
        S3Targets:
          - Path: s3://my-iot-data-bucket/raw/
      # 스키마 변경 시 행동 설정
      SchemaChangePolicy:
        UpdateBehavior: "UPDATE_IN_DATABASE"
        DeleteBehavior: "LOG"
      # 재크롤링 정책
      RecrawlPolicy:
        RecrawlBehavior: "CRAWL_NEW_FOLDERS_ONLY"

```

<br>

### 🔷 AWS Console에서 배포 (GUI 방식)
1. AWS Management Console 접속
2. 스택 생성 > 템플릿 파일 업로드 > .yaml 파일 업로드
3. 스택 이름 입력
4. IAM Role 등 확인 후, 스택 생성 버튼 클릭

<br>

### 🔷 AWS CLI에서 배포
```bash
aws cloudformation create-stack \
  --stack-name iot-crawler-stack \
  --template-body file://crawler.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

▶ 몇 분 이내에 Glue Crawler 생성됨!  
▶ CloudFormation Console 에서 리소스 상태 모니터링 가능  
▶ Glue Console에서 실제 크롤러가 잘 생성되었는지 확인  
▶▶ 필요한 경우 Glue Crawler 실행!


<br>

## ✅ Glue Crawler Update (update-stack)
템플릿을 수정하고, 변경사항을 AWS에 반영하고 싶을 때 사용

### 🔷 YAML 파일 수정
ex. 접두사 변경 (iot_ → sensor_)
```yaml
TablePrefix: sensor_
```

### 🔷 업데이트 명령어
```bash
aws cloudformation update-stack \
  --stack-name iot-crawler-stack \
  --template-body file://crawler.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```
🔥 업데이트 시, stack-name은 기존 스택 이름과 동일해야 함!


<br>

## ✅ Glue Crawler 변경 추적 (change-set)
변경 내용을 반영하기 전 변경사항을 미리 확인하고 싶을 때 사용
🔥 운영 환경에서 매우 중요!!

### 🔷 변경 세트 생성 (create-change-set)
```bash
aws cloudformation create-change-set \
  --stack-name iot-crawler-stack \
  --change-set-name crawler-change-1 \
  --template-body file://crawler.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

### 🔷 변경 세트 내용 확인 (describe-change-set)
```bash
aws cloudformation describe-change-set \
  --stack-name iot-crawler-stack \
  --change-set-name crawler-change-1
```

➕ 출력 예시
```json
{
  "Changes": [
    {
      "ResourceChange": {
        "Action": "Modify",
        "LogicalResourceId": "IoTCrawler",
        "ResourceType": "AWS::Glue::Crawler",
        "Replacement": "False",
        "Details": [
          {
            "Target": {
              "Attribute": "Properties",
              "Name": "TablePrefix",
              "RequiresRecreation": "Never"
            },
            "ChangeSource": "DirectModification",
            "CausingEntity": null
          }
        ]
      }
    }
  ]
}

```

### 🔷 적용 (Execute-change-set)
```bash
aws cloudformation execute-change-set \
  --stack-name iot-crawler-stack \
  --change-set-name crawler-change-1
```
▶ 해당 변경 내용이 기존 CloudFormation 스택에 반영됨  
▶ 기존 리소스들이 수정되거나 교체됨  
🔥 변경된 내용은 템플릿에 남지는 않고, AWS 내부에만 적용됨!!  
▶ `템플릿 파일 자체는 직접 로컬에서 관리하거나 Git 등으로 추적해야 함!!!`  


<br>

## ✅ Glue Crawler 삭제 (delete-stack)
스택과 내부 모든 리소스 삭제
Glue Crawler, 관련 DB, IAM Role 등 포함
```bash
aws cloudformation delete-stack --stack-name iot-crawler-stack
```

<br>

## ✅ 상태/로그/추적 확인
- describe-stacks: 현재 스택 상태 요약
- describe-stack-events: 최근 실행된 이벤트 로그
- list-stack-resources: 스택에 포함된 리소스 목록
- describe-change-set: 변경 세트 상세 확인
```bash
aws cloudformation describe-stack-events --stack-name iot-crawler-stack
```
