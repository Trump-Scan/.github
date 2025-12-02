# 저장소

## 1. 전체 저장소 구성

| 저장소 타입              | 선택            | 용도        | 컨테이너     |
|---------------------|---------------|-----------|----------|
| **RDBMS**           | Oracle (OCI)  | 영구 데이터 저장 | 신규 추가    |
| **Key-Value Store** | Redis         | 상태 관리, 캐싱 | 기존 사용    |
| **Vector DB**       | Qdrant        | 중복 검사     | 신규 추가    |
| **Message Queue**   | Redis Streams | 레이어 간 통신  | Redis 활용 |

**총 컨테이너**: 2개 (Redis, Qdrant)

---

## 2. 저장소별 역할

### 2.1 Oracle (RDBMS)
**역할**: 모든 데이터의 영구 저장소

**저장 데이터**:
- `raw_data` - 원본 수집 데이터
- `analysis_results` - 분석 결과 (요약, 산업 태그)
- `embedding_history` - 임베딩 벡터 이력
- `similarity_history` - 중복 판정 이력
- `feed_items` - 피드 아이템
- `feed_industries` - 피드-산업 매핑

**특징**:
- 영구 보존
- 감사 추적
- 통계 분석
- 재처리 지원

---

### 2.2 Redis (Key-Value Store + Message Queue)

#### Key-Value Store 역할
**저장 데이터**:
- Checkpoint (수집 위치)
- 처리 상태 (레이어별)
- API 캐시
- Rate Limiting 카운터
- 실시간 연결 정보

**특징**:
- TTL 자동 관리
- 빠른 읽기/쓰기
- 레이어 간 상태 공유

#### Message Queue 역할 (Redis Streams)
**Stream 구성**:
- `queue:analysis` - 수집 → 분석
- `queue:deduplication` - 분석 → 중복제거
- `queue:feed` - 중복제거 → 피드

**특징**:
- Consumer Group 지원
- DLQ 직접 구현 (30-50줄)
- 처리량: 360개/일

---

### 2.3 Qdrant (Vector DB)
**역할**: 중복 검사 전용

**저장 데이터**:
- 임베딩 벡터 (384차원)
- 메타데이터 (ID, 채널, 타임스탬프)
- 고유 데이터만 저장 (중복은 제외)

**특징**:
- TTL 48시간 자동 삭제
- 코사인 유사도 검색
- 웹 UI 제공
- 저장량: 약 216개

**임베딩 모델**:
- all-MiniLM-L12-v2 또는 all-MiniLM-L6-v2
- 차원: 384

---

## 3. 레이어별 저장소 매핑

```
┌─────────────────┐
│  수집 레이어     │
├─────────────────┤
│ Oracle: raw_data│
│ Redis: checkpoint│
│ Redis: status   │
└────────┬────────┘
         │ Redis Streams (queue:analysis)
         ↓
┌─────────────────────────┐
│    분석 레이어          │
├─────────────────────────┤
│ Oracle: analysis_results│
│ Redis: status           │
└────────┬────────────────┘
         │ Redis Streams (queue:deduplication)
         ↓
┌───────────────────────────┐
│   중복 제거 레이어        │
├───────────────────────────┤
│ Qdrant: 벡터 검색         │
│ Oracle: embedding_history │
│ Oracle: similarity_history│
│ Oracle: feed_items        │
│ Oracle: feed_tags         │
│ Redis: status             │
└───────────────────────────┘
```

---

## 4. Docker Compose 구성

```yaml
version: '3.8'

services:
  # Redis (Key-Value Store + Message Queue)
  redis:
    image: redis:7-alpine
    container_name: trumpscan-redis
    ports:
      - "6379:6379"
    volumes:
      - ./data/redis:/data
    command: redis-server --appendonly yes
    restart: unless-stopped
    networks:
      - trumpscan-network

  # Qdrant (Vector DB)
  qdrant:
    image: qdrant/qdrant:latest
    container_name: trumpscan-qdrant
    ports:
      - "6333:6333"    # REST API
      - "6334:6334"    # gRPC
    volumes:
      - ./data/qdrant:/qdrant/storage
    restart: unless-stopped
    networks:
      - trumpscan-network

networks:
  trumpscan-network:
    driver: bridge

volumes:
  redis-data:
  qdrant-data:
```

**접속 정보**:
- Redis: `localhost:6379`
- Qdrant REST API: `http://localhost:6333`
- Qdrant 웹 UI: `http://localhost:6333/dashboard`
- Oracle: OCI 설정 참조

---

## 5. 데이터 플로우

```
1. 데이터 수집
   ↓ (Oracle: raw_data 저장)
   ↓ (Redis: checkpoint 업데이트)
   ↓ (Redis Streams: queue:analysis)

2. 분석
   ↓ (Oracle: analysis_results 저장)
   ↓ (Redis Streams: queue:deduplication)

3. 중복 제거 + 피드 저장
   ↓ (Qdrant: 벡터 검색)
   ↓ (Oracle: embedding_history, similarity_history 저장)
   ↓ (Oracle: feed_items, feed_industries 저장)
   ↓ (Redis Streams: queue:api)

4. API 응답
   ↓ (Redis: 캐시에서 조회)
   ↓ (Oracle: 데이터베이스 조회)
```

---

## 6. 선택 이유 요약

### Oracle
- ✅ 기존 인프라 활용
- ✅ 엔터프라이즈급 안정성
- ✅ 복잡한 쿼리 지원

### Redis (Key-Value + MQ)
- ✅ 기존 인프라 활용
- ✅ 이중 역할 (상태 관리 + 메시징)
- ✅ 컨테이너 절약
- ✅ 빠른 성능

### Qdrant
- ✅ 간단한 설정 (컨테이너 1개)
- ✅ TTL 자동 삭제
- ✅ 웹 UI 제공
- ✅ 우수한 성능

### Redis Streams (MQ)
- ✅ 컨테이너 추가 불필요
- ✅ DLQ 구현 가능
- ✅ 충분한 처리량

---

## 7. 리소스 요약

### 데이터 규모
- 일일 수집: 120개
- 일일 메시지: 360개
- Qdrant 저장: 216개 (48시간 TTL)
- Oracle: 누적 저장

### 컨테이너 리소스
| 컨테이너 | CPU | 메모리 | 디스크 |
|----------|-----|--------|--------|
| Redis | 0.1 | 256MB | 1GB |
| Qdrant | 0.2 | 512MB | 2GB |
| **합계** | **0.3** | **768MB** | **3GB** |

**맥북 에어 M2 16GB에서 여유롭게 실행 가능**

---

## 8. 모니터링 포인트

### Redis
```bash
# Redis CLI
redis-cli info stats
redis-cli xlen queue:analysis
```

### Qdrant
- 웹 UI: http://localhost:6333/dashboard
- 벡터 수, 검색 속도 확인

### Oracle
- OCI Console 또는 SQL Developer
- 테이블 크기, 쿼리 성능 모니터링

---

## 9. 백업 전략

| 저장소 | 백업 대상 | 백업 주기 | 방법 |
|--------|-----------|-----------|------|
| Oracle | 전체 데이터 | 일 1회 | OCI 자동 백업 |
| Redis | 데이터 스냅샷 | 일 1회 | RDB/AOF 백업 |
| Qdrant | 벡터 데이터 | 불필요 | TTL 48시간 (휘발성) |

---

## 10. 확장 시나리오

### 데이터 10배 증가 (일 1,200개)
- ✅ Oracle: 문제없음
- ✅ Redis: 문제없음
- ✅ Qdrant: 문제없음
- ✅ Redis Streams: 문제없음

### 데이터 100배 증가 (일 12,000개)
- ✅ Oracle: 파티셔닝 고려
- ✅ Redis: Cluster 고려
- ✅ Qdrant: 단일 노드 가능
- ⚠️ Redis Streams → RabbitMQ 고려

### 대규모 (일 100만개+)
- ⚠️ Oracle → 샤딩 또는 PostgreSQL
- ⚠️ Redis → Redis Cluster
- ⚠️ Qdrant → 클러스터링
- ⚠️ Redis Streams → Kafka

---

## 11. 비용 분석

| 항목 | 월 비용 | 비고 |
|------|---------|------|
| Oracle (OCI) | 기존 사용 | 기존 인프라 |
| Redis | $0 | 자체 호스팅 |
| Qdrant | $0 | 자체 호스팅 |
| **총계** | **$0** | **추가 비용 없음** |

**로컬 개발**: 완전 무료  
**프로덕션 배포**: 서버 비용만 추가

---

## 12. 결론

**4개 저장소 조합은 MVP 빠른 구축과 향후 확장성을 모두 만족합니다.**

### 주요 장점
✅ **최소 인프라**: 컨테이너 2개만 추가  
✅ **무료**: 추가 비용 없음  
✅ **간단한 설정**: 5분 안에 환경 구축  
✅ **확장 가능**: 데이터 100배 증가까지 대응  
✅ **검증된 기술**: 프로덕션 레벨 안정성

### 역할 분담
- **Oracle**: 영구 저장, 복잡한 쿼리
- **Redis**: 빠른 상태 관리, 메시징
- **Qdrant**: 중복 검사
- 각 저장소가 명확한 책임 분리