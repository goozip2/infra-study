# ✔️ Amazon OpenSearch Service
검색 엔진
>문서 기반 검색, 인덱스 개념, 검색 쿼리 예시

`OpenSearch 클러스터를 쉽게 배포, 운영, 확장할 수 있는 관리형 서비스`

https://docs.aws.amazon.com/ko_kr/opensearch-service/latest/developerguide/what-is.html

: `설정`, `인스턴스 유형`, `인스턴스 수`, `스토리지 리소스`를 갖고 있는 클러스터  
: 로그 분석, 실시간 애플리케이션 모니터링, 클릭 스트림 분석 등을 수행하는 오픈소스 `검색` 및 `분석` 엔진  
: OpenSearch는 단순한 저장소가 아니라 검색 + 분석이 가능한 검색엔진 기반 데이터 플랫폼  


<br>

## ✅ OpenSearch Service 핵심 기능
### 🔷 검색 엔진
구조화된 데이터 또는 반구조화된 데이터에 대해 빠른 Full-Text 검색 가능

### 🔷 분석 플랫폼
수집한 로그/지표 등을 실시간 집계, 시각화 가능 (Kibana 대체: DashBoards 제공)

### 🔷 저장소 역할
내부적으로 데이터를 디스크에 저장하는 엔진 구조 사용

<br>

### ➕ 흐름 정리
- OpenSearch 클러스터 노드들은 EC2 인스턴스처럼 동작
- EBS 볼륨이 각 노드에 연결되어 있어 데이터와 색인 파일을 물리적으로 저장
- EBS는 AWS에서 제공하는 블록 스토리지 서비스로 디스크 역할 수행




<br>

## ✅ OpenSearch Service 제공 기능
### 🔷 크기 조정
- CPU, 메모리, 스토리지 용량 구성
- 데이터 노드 지원

### 🔷 보안
- IAM 액세스 제어
- VPC 보안 그룹 사용하는 쉬운 통합
- 저장된 데이터의 암호화 및 노드 간 암호화
- OpenSearch 대시보드에 대한 접근 인증
- 인덱스 수준, 문서 수준 및 필드 수준 보안
- 감사 로그
- DashBoard 멀티테넌시


### 🔷 유연성
- BI 애플리케이션과의 통합을 위한 SQL 지원
- 검색 결과 개선을 위한 사용자 지정 패키지


### 🔷 타 서비스와의 통합
- OpenSearch 대시보드를 사용한 데이터 시각화
- OpenSearch Service 도메인 지표 및 설정 경보 모니터링 위한 Amazon CloudWatch와의 통합
- OpenSearch Service 도메인에 AWS CloudTrail 구성, API 호출 감사를 위한 통합
- 스트리밍 데이터를 OpenSearch Serivce로 로드하기 위해 S3, Kinesis, DynamoDB와의 통합
- 데이터가 특정 임계값을 초과하는 경우 Amazon SNS 알림


<br>

## ✅ OpenSearch Service 구조

```
📦 OpenSearch Cluster
│
├─ 🌐 REST API Endpoint (클러스터 접근 포인트)
│
├─ 📚 Index (인덱스)
│   ├─ 📄 Document 1
│   ├─ 📄 Document 2
│   └─ ...
│
├─ ⚙️ Node (서버)
│   ├─ 🧠 Master Node (클러스터 상태, 설정 관리)
│   ├─ 💾 Data Node (데이터 저장, 질의 처리)
│   └─ 🧮 Ingest Node (데이터 파이프라인 전처리)
│
└─ 🧱 Shard & Replica
    ├─ Primary Shard (실제 데이터 분산 저장)
    └─ Replica Shard (장애 대비 복제본)

```
- `Index`: 관계형 DB 테이블과 유사한 개념, document 저장하는 논리적 단위
- `Document`: JSON 기반 데이터 단위. RDB의 Row와 유사
- `Shard`: Index를 물리적으로 나눈 단위. 여러 노드에 분산 저장됨
- `Replica`: Shard의 복사본. 장애 대비, 읽기 성능 향상 목적
- `Node`: 클러스터를 구성하는 물리/논리 서버
- `Cluster`: 여러 노드가 연결되어 하나의 시스템처럼 동작하는 집합

<br>

## ✅  제공하는 상세 기능
- `Search API`: 다양한 질의, 정렬, 필터링 제공
- `Aggregation`: 집계 분석 (sum, avg, terms, histogram 등)
- `Ingest Pipline`: 데이터 수집/정제 처리 (timestamp 추가, 필드 추출)
- `Alerting`: 조건 기반 알림 생성 (log에 error 발생 시 이메일 알림)
- `DashBoards`: 시각화 UI 제공 (kinesis 오픈소스 기반)
- `Security`: IAM 인증, 도메인 접근 제한, IP 허용 등
- `Serverless`: 클러스터 구성 없이 사용할 수 있는 서버리스 방식도 제공 중

<br>

## ✅ 서버 리스 방식
- 클러스터 관리 없이 빠르게 로그 검색, 분석 플랫폼 구축
- 인덱스/도메인 대신 Collection 단위로 관리
- 오토 스케일: 요청 부하에 따라 컴퓨팅 리소스 자동 확장/축소
- 로그 분석, 보안 이벤트 감지, IoT 스트림 검색 등

### 🔷 OpenSearch Serverless 구성 요소
- `Collection`: 하나 이상의 인덱스를 포함하는 논리적 단위 (핵심 단위)
- `Encryption Policy`: 암호화 정책
- `Access Policy`: IAM 기반 접근 제어 정책
- `Data Access Role`: Firehose, Lambda 등에서 데이터를 수집할 때 사용하는 역할
- `OCU (OpenSearch Comput Unit)`: 서버리스 처리 성능 단위 (자동 확장)

```
[IoT / App / Log Data]
        ↓
  [Kinesis Firehose / Lambda / SDK]
        ↓
 🔐 [Access Role + Policy]
        ↓
📦 [OpenSearch Serverless Collection]
        ↓
 🔍 Query → 🔄 Index → 📊 Dashboards

```

## ✅ IaC로 관리
- Collection
- Domain (EC2 기반 도메인 설정)
- Index Template: 인덱스 매핑, 셋팅 자동 정의
- Dashboard: JSON, YAML 정의로 시각화 저장 가능
- Alert Rule: CPU, 트래픽 등 조건 기반 경고 설정
- Access Policy: IAM 기반 접근 제어 정책
- Data Source 설정: 연동하는 리소스도 함께 IaC화 가능

<br>

## ✅ OpenSearch 도메인 생성 (IaC)
### 🔷 AWS Console에서 도메인 생성 (클러스터 구성)

```yaml
OpenSearchDomain:
  Type: AWS::OpenSearchService::Domain
  Properties:
    DomainName: my-domain
    EngineVersion: "OpenSearch_2.11"
    ClusterConfig:
      InstanceType: t3.small.search
      InstanceCount: 1
    # EBS: OpenSearch 클러스터의 DB 파일이 저장되는 디스크
    EBSOptions:
      EBSEnabled: true
      VolumeSize: 10
    AccessPolicies:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Principal: "*"
          Action: "es:*"
          Resource: "*"

```

<br>

### 🔷 데이터 인덱싱 (PUT or Bulk API)
> INSERT + 자동 색인
: 문서를 클러스터에 저장하면서 검색 가능한 구조로 색인을 구축하는 과정

- 검색 엔진이 이해할 수 있는 형태로 구조화된 저장
- 색인(inverted index) 생성

1. 각 필드 분석
2. inverted Index 생성
3. EBS(저장소)에 저장

```python
import requests
import json

# OpenSearch에 저장할 데이터 문서(document)
doc = {
    "name": "sensor-A1",
    "temperature": 27.5,
    "timestamp": "2025-07-08T08:00:00Z"
}

# OpenSearch REST API에 HTTP PUT 요청을 보내는 부분
response = requests.put(
    "https://search-my-domain/_doc/1",
    headers={ "Content-Type": "application/json" },
    data=json.dumps(doc)
)
print(response.status_code, response.text)

```
실제 문서 데이터와 인덱스 정보는 클러스터 내 노드들이 마운트한 EBS 볼륨(디스크)에 저장됨


➕ 실시간 데이터 인덱싱 구성 방법
- Lambda + Event Trigger
- API 서버 연동: 실시간으로 OpenSearch API 호출
- Streaming 서비스 활용: firehose → OpenSearch
  - Kinesis Firehose가 OpenSearch에 데이터 전달 시, 내부적으로 OpenSearch의 REST API를 호출하여 데이터를 적재

<br>

### 🔷 데이터 조회 (Search API)
>SELECT
1. 전체 문서 조회
```bash
curl -XGET 'https://your-endpoint/my-index/_search' -H 'Content-Type: application/json' -d '{
  "query": { "match_all": {} }
}'
```

2. 조건 검색
```bash
curl -XGET 'https://your-endpoint/my-index/_search' -H 'Content-Type: application/json' -d '{
  "query": {
    "match": { "name": "sensor-A1" }
  }
}'

```

<br>

### 🔷 Kibana (OpenSearch Dashboards) 연동

### ➕ EX. 로그 저장 + 검색
- CloudWatch Logs: 로그 수집
- Kinesis Firehose: 로그 전송 + JSON + OpenSearch 적재
- OpenSearch Dashboards: 로그 검색 및 이상 탐지 시각화

```
[ IoT Device / App Logs ]
         ↓
  [ CloudWatch Logs ]
         ↓
[ Kinesis Firehose (Delivery Stream) ]
         ↓
  [ Amazon OpenSearch Service ]
         ↓
[ OpenSearch Dashboards (시각화) ]
``` 
