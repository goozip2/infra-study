# ✔️ AWS 리소스 제어 방법
## ✅ AWS Console (Web UI)
AWS 웹사이트에서 마우스로 직접 설정
- 초기 테스트
- 소규모 설정

😊Pros: 직관적, 즉각 피드백  
😭Cons: 반복잡업 불편, 자동화 불편

<br>

## ✅ AWS CLI
터미널에서 명령어로 AWS 조작
- 간단한 작업 자동화
- 스크립트 작성
- 빠른 리소스 확인

😊Pros: 빠르고 간단, 스크립트화 가능  
😭Cons: 복잡한 로직 구현 어려움

<br>

## ✅ AWS SDK (boto3, Java SDK)
프로그래밍 언어에서 AWS API 호출
복잡한 자동화 코드 작성
- 복잡한 로직 자동화
- 대규모 파이프라인
- 이벤트 기반 처리

😊Pros: 프로그래밍 가능성 무한대, 복잡한 흐름 구현 가능  
😭Cons: 개발자 역량 요구, 초기 학습 필요

<br>

## ✅ IaC (CloudFormation, Terraform, CDK)
선언적 인프라 구성 관리
인프라 전체 구성 및 배포 관리
- 인프라 반복 배포
- 팀 협업
- 버전 관리 및 복원

😊Pros: 코드로 인프라 관리, 재사용성 우수  
😭Cons: 러닝 커브 있음, 실시간 조작 어려움

3가지 방법
- AWS CloudFormation (YAML/JSON)
- Terraform (HCL)
- AWS CDK (Python, TypeScript 등 프로그래밍 언어)



<br>

----


<br>

# ✔️ IaC 리소스 호출 방식
## ✅ Lambda + EventBridge (정기 실행)
- 주기적 실행 (스케줄링)
- 서비리스 + 자동 실행

### 🔷 EX
```
[EventBridge 스케줄 (매일 자정)]
        ↓
[AWS Lambda]
        ↓
[Glue Crawler 실행 or Athena 쿼리 실행]
```
- IaC로 리소스 선언
- Lambda로 실행
- Cron 처럼 동작

<br>


## ✅ Lambda + Event 기반 트리거 (S3, IoT, Kafka 등)
- 실시간 데이터 처리
- 이벤트 기반으로 실시간 자동화 (센서 → Lambda → 처리)

### 🔷 EX
```
[IoT 디바이스 → MQTT / S3 업로드]
        ↓ (Trigger)
[AWS Lambda]
        ↓
[데이터 정제 → S3 저장 or Glue 호출]
```
- 실시간 자동 호출
- 이벤트에 반응하여 Lambda가 자동 실행됨


<br>


## ✅ CI/CD 파이프라인
CodePipline, GitHub Action
- IaC 배포, 테트스, 쿼리 실행 자동화
- IaC 기반 인프라 구성 자동화의 핵심 도구

```
[코드 푸시 or PR 머지]
        ↓
[GitHub Actions or CodePipeline]
        ↓
[CloudFormation 템플릿 배포 (IaC)]
        ↓
[테스트 데이터 쿼리 or Lambda 자동 호출]
```
- 개발자 commit → 자동 배포 + 실행
- 배포 자동화, 테스트 자동화
