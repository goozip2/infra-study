# ✔️ Amazon Redis
`인메모리(In-memory)` 기반의 초고속 `Key-Value` 저장소

https://aws.amazon.com/ko/elasticache/what-is-redis/

<br>

## ✅ 특징 요약
- 데이터 저장: `RAM`에 저장
- 형태: `Key-Value` 구조
- 속도: `마이크로sec` 단위
- 지속성: 디스크 `백업` 가능
  - RDB (snapshot)
  - AOF (Append Only File)  
  ▶ 원할 경우 영속성 보장 가능 (완전 휘발성 X)
- `데이터 타입`: String, List, Set, Hash, Sorted Set 등
![](https://velog.velcdn.com/images/goozip2/post/00188ea2-0b7e-4042-bfac-5fb3de395547/image.png)


<br>

## ✅ Redis 주요 사용 사례
- `캐시`: DB 조회 결과를 Redis에 저장하여 빠르게 응답
- `세션 저장소`: 로그인 상태 저장
- `메시지 큐`: Pub/Sub 또는 List 활용
- `실시간 랭킹`: Sorted Set 활용
- `Rate Limiting`: 초당 요청 수 제어
- `tream 처리`: Redis Streams 이용 (Kafka 대체 가능성 존재)

<br>

### 🔷 Redis를 IoT 수집에 사용하는 이유
- 메모리 기반이기 때문에 수십만 TPS도 가능
- 수집된 데이터를 DB에 보내기 전 버퍼링
- Key, List, Hash, SortedSet 등 다양한 방식으로 저장
- 메시지 큐처럼 처리 가능 (IoT에서 중요)
- TTL 설정으로 일정 시간 지나면 자동 삭제 가능

<br>

## ✅ 실습: Redis 간단 연결
### 🔷 Redis 실행 (Docker 사용)
```bash
docker run --name redis -p 6379:6379 -d redis
```
별도 설치 없이 Redis 서버를 로컬에서 바로 띄울 수 있다.

### 🔷 Python 코드로 Redis 연결
```bash
pip install redis
```

```python
import redis

# Redis 서버 연결
r = redis.StrictRedis(host='localhost', port=6379, decode_responses=True)

# IoT 데이터 저장
iot_data = {
    "device_id": "sensor-A1",
    "temperature": 27.5,
    "timestamp": "2025-07-08T09:00:00Z"
}

# 1. 문자열로 저장 (Key-Value)
r.set("sensor-A1:latest", str(iot_data))

# 2. 리스트에 저장 (시간순 적재)
r.rpush("sensor-A1:data", str(iot_data))

# 3. 해시로 저장
r.hset("sensor-A1:hash", mapping=iot_data)

# 저장된 데이터 조회
print("Latest:", r.get("sensor-A1:latest"))
print("All Logs:", r.lrange("sensor-A1:data", 0, -1))
print("Hash:", r.hgetall("sensor-A1:hash"))

```

<br>

## ✅ 실습: Redis를 Queue로 사용하는 소비자 구조

```
[IoT 디바이스]
    ↓ MQTT
[AWS IoT Core or 직접 수신 서버]
    ↓ Python Lambda / Flask / FastAPI 등에서 수신
[Redis]
    → 임시 저장 (Queue, Buffer, 캐시)
    ↓
[Batch or Worker Process]
    → Timestream, S3, OpenSearch 등으로 전달
```

- Redis를 버퍼/큐 역할로 사용하여,
  - 실시간 수신은 Redis에 맡기고,
  - 분석/전송은 나중에 처리하는 구조 가능

```python
# Redis에서 IoT 데이터 하나씩 꺼내 처리
while True:
    data = r.lpop("sensor-A1:data")  # FIFO
    if data:
        print("처리 중:", data)
        # → Timestream에 저장하거나, DB 전송 처리
    else:
        break

```

<br>

## ✅ Redis 사용 시 주의사항
### 🔷 메모리 제한
Redis는 RAM 기반이기 때문에, 데이터가 너무 많으면 OOM(Out Ot Memory)로 터진다.

#### 🔸 TTL (Time To Live)
Redis 키에 만료 시간을 설정하여, 자동으로 삭제되도록 한다.  
IoT 센서 실시간 데이터 저장 → 일정 시간 후 자동 소멸
```python
r.set("temp:device-A1", 27.5, ex=300)  # 5분 후 삭제(ex)
```



#### 🔸 LRU (Least Recently Used)
Redis는 메모리 부족 시, 가장 덜 사용된 데이터 자동 삭제 가능하다.  
`maxmemory`, `maxmemory-policy` 설정
```bash
# redis.conf 예시
maxmemory 256mb
maxmemory-policy allkeys-lru
```
![](https://velog.velcdn.com/images/goozip2/post/36ef6674-da73-4ac5-b655-b534c032aa0b/image.png)

Redis를 Cache 처럼 사용 시, allkeys-lru 추천!

<br>

### 🔷 지속성
Disk Backup 되지만, 주요 DB로는 부적절하다.

#### 🔸 RDB (Snapshotting)
- 특정 주기마다 전체 데이터를 스냅샷으로 저장
- dump.rdb 파일 생성
- redis.conf에서 설정

▶ 빠르고 간단, 복구는 전체 상태 기준

#### 🔸 AOF (Append Only File)
- 모든 쓰기 명령을 순차적으로 기록
- appendonly.aof 파일 생성

▶ 느리지만 정확하고 최신 상태까지 복구 가능  
▶ 실시간성 중요하거나, 복구 정밀도가 중요한 경우 사용


<br>

### 🔷 보안

기본 Redis는 인증 없이 누구나 접근 가능하다.
requirepass, 방화벽 설정 필수!!












