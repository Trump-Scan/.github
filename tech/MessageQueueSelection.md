# Message Queue 선택 문서

**작성일**: 2025-11-12  
**프로젝트**: Trump Scan Service  
**결정**: Redis Streams (기존 Redis 활용)

---

## 1. 선택 결과

**Message Queue: Redis Streams**

---

## 2. 프로젝트 제약사항

### 환경
- 개발: 맥북 에어 M2, 16GB RAM
- 배포: Docker 기반
- 개발자: 1인 (백엔드)
- 목표: MVP 빠른 구축

### 비용
- 무료 (자체 호스팅)
- 외부 API 호출 없음

### 기존 인프라
- Oracle (RDBMS)
- **Redis (Key-Value Store) - 이미 사용 중**

---

## 3. 기술 요구사항

### 메시지 처리량
- 일일 메시지: 360개 (120개 × 3 레이어)
- 초당 처리: 약 0.004개 (매우 낮음)

### 메시지 특성
- 순서 보장: 불필요
- 메시지 크기: 중간 (수백 바이트 ~ 수 KB)
- 유실 허용: 불가
- Dead Letter Queue: 필요

### 레이어 간 통신
```
수집 레이어 → [MQ] → 분석 레이어
분석 레이어 → [MQ] → 중복 제거 레이어
중복 제거 레이어 → [MQ] → API 레이어
```

---

## 4. 평가 우선순위

1. **설정 간편함** (가장 중요)
2. **개발 편의성**
3. **성능**
4. **안정성/문서**

---

## 5. 후보 비교

| 항목 | Redis Streams | RabbitMQ | Kafka |
|------|---------------|----------|-------|
| 컨테이너 추가 | ❌ (기존 활용) | ✅ (1개) | ✅ (2개+) |
| 설정 복잡도 | 없음 | 간단 | 복잡 |
| DLQ 지원 | 직접 구현 | 네이티브 | 직접 구현 |
| 관리 UI | ❌ | ✅ | △ |
| 성능 (360개/일) | 여유 | 여유 | 극심한 과소 |
| MVP 적합 | ✅ | ✅ | ❌ |

---

## 6. Redis Streams 선택 이유

### 핵심 이유
1. **인프라 단순화** (컨테이너 추가 불필요)
2. **설정 작업 없음** (기존 Redis 그대로 활용)
3. **빠른 개발** (MVP에 최적)
4. **우수한 성능** (메모리 기반)
5. **충분한 기능** (Consumer Group, ACK 지원)

### 결정적 요소
- **이미 Redis 사용 중**: 인프라 중복 없음
- **MVP 우선**: 빠른 구축이 최우선
- **낮은 처리량**: 360개/일 수준에서 전문 MQ 불필요

### 트레이드오프
- ⚠️ DLQ 직접 구현 필요 (30-50줄)
- ⚠️ 관리 UI 없음
- ✅ 하지만 인프라 단순화로 충분히 보상

---

## 7. 제외된 후보

### RabbitMQ
- 장점: DLQ 네이티브, 관리 UI, 안정성
- 제외 이유: 컨테이너 추가 필요, MVP에 과함

### Kafka
- 장점: 최고 성능, 확장성
- 제외 이유: 극심한 오버스펙, 복잡한 설정, 리소스 과다

---

## 8. Redis Streams 기술 스펙

### 기존 Redis 활용
```yaml
# docker-compose.yml (변경 없음)
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - ./redis_data:/data
```

### Stream 구조
```
Stream: queue:analysis
  ├─ Consumer Group: analysis-workers
  │   ├─ Consumer: worker-1
  │   └─ Consumer: worker-2
  └─ DLQ: queue:analysis:dlq

Stream: queue:deduplication
  ├─ Consumer Group: dedup-workers
  │   └─ Consumer: worker-1
  └─ DLQ: queue:deduplication:dlq

Stream: queue:api
  ├─ Consumer Group: api-workers
  │   └─ Consumer: worker-1
  └─ DLQ: queue:api:dlq
```

### Python 클라이언트
```bash
pip install redis
```

### 기본 사용 예시

#### Producer
```python
import redis
import json

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# 메시지 전송
message = {
    "data_id": "uuid-1234",
    "content": "Trump statement...",
    "timestamp": "2025-11-12T10:00:00Z"
}

redis_client.xadd(
    "queue:analysis",
    {"payload": json.dumps(message)}
)
```

#### Consumer (기본)
```python
import redis
import json

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Consumer Group 생성 (최초 1회)
try:
    redis_client.xgroup_create("queue:analysis", "analysis-workers", id='0', mkstream=True)
except redis.exceptions.ResponseError:
    pass  # 이미 존재

# 메시지 수신
while True:
    messages = redis_client.xreadgroup(
        groupname="analysis-workers",
        consumername="worker-1",
        streams={"queue:analysis": ">"},
        count=1,
        block=5000  # 5초 대기
    )
    
    for stream, message_list in messages:
        for message_id, data in message_list:
            payload = json.loads(data["payload"])
            
            try:
                # 메시지 처리
                process(payload)
                
                # ACK
                redis_client.xack("queue:analysis", "analysis-workers", message_id)
                
            except Exception as e:
                print(f"Error: {e}")
                # 재시도 로직은 아래 DLQ 구현 참고
```

---

## 9. DLQ 구현 (30-50줄)

### 전체 구현 코드

```python
import redis
import json
import time
from typing import Dict, Any

# 설정
MAX_RETRY = 3
RETRY_DELAY = 60  # 초

class RedisStreamConsumer:
    def __init__(self, redis_client, stream_name, group_name, consumer_name):
        self.redis = redis_client
        self.stream = stream_name
        self.group = group_name
        self.consumer = consumer_name
        self.dlq_stream = f"{stream_name}:dlq"
        
        # Consumer Group 생성
        try:
            self.redis.xgroup_create(self.stream, self.group, id='0', mkstream=True)
        except redis.exceptions.ResponseError:
            pass
    
    def consume(self, process_func):
        """메시지 소비 + DLQ 처리"""
        while True:
            messages = self.redis.xreadgroup(
                groupname=self.group,
                consumername=self.consumer,
                streams={self.stream: ">"},
                count=1,
                block=5000
            )
            
            if not messages:
                continue
            
            for stream, message_list in messages:
                for message_id, data in message_list:
                    self._process_with_retry(message_id, data, process_func)
    
    def _process_with_retry(self, message_id: str, data: Dict, process_func):
        """재시도 + DLQ 로직"""
        payload = json.loads(data.get("payload", "{}"))
        retry_count = int(data.get("retry_count", 0))
        
        try:
            # 실제 처리
            process_func(payload)
            
            # 성공 시 ACK
            self.redis.xack(self.stream, self.group, message_id)
            
        except Exception as e:
            print(f"Error processing message {message_id}: {e}")
            
            if retry_count >= MAX_RETRY:
                # 최대 재시도 초과 → DLQ로 이동
                self._move_to_dlq(message_id, data, str(e))
            else:
                # 재시도
                self._retry(data, retry_count)
            
            # 원본 메시지 ACK (재시도용 새 메시지 생성했으므로)
            self.redis.xack(self.stream, self.group, message_id)
    
    def _retry(self, data: Dict, retry_count: int):
        """재시도 큐에 추가"""
        data["retry_count"] = retry_count + 1
        data["retry_at"] = time.time() + RETRY_DELAY
        
        self.redis.xadd(self.stream, data)
    
    def _move_to_dlq(self, message_id: str, data: Dict, error: str):
        """DLQ로 이동"""
        dlq_data = {
            **data,
            "original_message_id": message_id,
            "error": error,
            "failed_at": time.time()
        }
        
        self.redis.xadd(self.dlq_stream, dlq_data)
        print(f"Message {message_id} moved to DLQ")

# 사용 예시
def process_analysis(payload):
    """분석 처리 로직"""
    print(f"Processing: {payload}")
    # 실제 처리 로직
    if some_error_condition:
        raise Exception("Processing failed")

# 소비자 실행
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
consumer = RedisStreamConsumer(
    redis_client=redis_client,
    stream_name="queue:analysis",
    group_name="analysis-workers",
    consumer_name="worker-1"
)

consumer.consume(process_analysis)
```

### DLQ 메시지 조회

```python
# DLQ 메시지 확인
dlq_messages = redis_client.xrange("queue:analysis:dlq", count=10)

for message_id, data in dlq_messages:
    print(f"Failed message: {message_id}")
    print(f"Error: {data['error']}")
    print(f"Payload: {data['payload']}")
```

### DLQ 메시지 재처리

```python
# DLQ에서 메시지 가져와서 재처리
dlq_messages = redis_client.xrange("queue:analysis:dlq", count=10)

for message_id, data in dlq_messages:
    # 원본 큐로 재전송
    retry_data = {
        "payload": data["payload"],
        "retry_count": 0
    }
    redis_client.xadd("queue:analysis", retry_data)
    
    # DLQ에서 삭제
    redis_client.xdel("queue:analysis:dlq", message_id)
```

---

## 10. 레이어별 Stream 구성

| 레이어 | Stream 이름 | Consumer Group | DLQ Stream |
|--------|-------------|----------------|------------|
| 수집 → 분석 | `queue:analysis` | `analysis-workers` | `queue:analysis:dlq` |
| 분석 → 중복제거 | `queue:deduplication` | `dedup-workers` | `queue:deduplication:dlq` |
| 중복제거 → API | `queue:api` | `api-workers` | `queue:api:dlq` |

---

## 11. 모니터링

### Stream 상태 확인
```python
# Stream 길이
length = redis_client.xlen("queue:analysis")
print(f"Queue length: {length}")

# Consumer Group 정보
info = redis_client.xinfo_groups("queue:analysis")
print(info)

# Pending 메시지 (처리 중인 메시지)
pending = redis_client.xpending("queue:analysis", "analysis-workers")
print(f"Pending messages: {pending['pending']}")
```

### DLQ 알림
```python
# DLQ에 메시지가 쌓이면 알림
dlq_length = redis_client.xlen("queue:analysis:dlq")
if dlq_length > 10:
    print(f"WARNING: DLQ has {dlq_length} messages!")
    # 슬랙 알림, 이메일 등
```

---

## 12. 장점 요약

✅ **설정 없음**: 기존 Redis 그대로 사용  
✅ **인프라 단순**: 컨테이너 추가 불필요  
✅ **빠른 개발**: 30-50줄로 DLQ 구현  
✅ **충분한 성능**: 360개/일 처리 여유  
✅ **낮은 운영 부담**: Redis 하나만 관리

---

## 13. 향후 고려사항

### 확장 시나리오
- **메시지 증가 (일 1만건+)**: Redis Streams 그대로 사용 가능
- **복잡한 라우팅 필요**: RabbitMQ 고려
- **대규모 스트리밍 (일 100만건+)**: Kafka 고려

### 전환 비용
- Redis Streams → RabbitMQ: 낮음 (인터페이스 유사)
- 필요 시 레이어별로 점진적 전환 가능

---

## 14. 결론

**Redis Streams는 기존 인프라를 활용하여 MVP를 빠르게 구축할 수 있는 최적의 선택입니다.**

- 설정 간편: ✅
- 개발 편의: ✅
- 성능: ✅
- 비용: ✅ (추가 비용 없음)

DLQ 직접 구현 부담(30-50줄)보다 인프라 단순화 이점이 훨씬 큽니다.

---

**문서 버전**: 1.0  
**이전 단계**: Vector DB 선택 (Qdrant)  
**완료**: 4개 저장소 중 3개 확정 (Oracle, Redis, Qdrant)