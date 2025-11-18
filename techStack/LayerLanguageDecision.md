# 레이어별 프로그래밍 언어 결정

## 최종 결정

| 레이어 | 언어 | 비고 |
|--------|------|------|
| 데이터 수집 | Python | 웹 스크래핑, RSS 파싱 |
| 분석 | Python | LLM API 호출, JSON 처리 |
| 중복 제거 | Python | SentenceTransformers 임베딩 |
| 피드 생성 | Node.js | 피드 항목 구성 및 저장 |
| API | Node.js | Express 또는 Fastify |

**총 언어 수**: 2개 (Python, Node.js)

---

## 결정 근거

### 전체 구성: Python + Node.js

**선택 이유**
1. **언어 수 최소화**: 2개 언어로 관리 부담 감소
2. **역할 분리 명확**: 데이터 처리(Python) vs 서빙(Node.js)
3. **각 언어의 강점 활용**: Python의 ML/데이터 생태계, Node.js의 비동기 I/O
4. **타입 공유**: 피드 생성과 API가 동일 언어로 타입 정의 공유 가능

**장점**
- 일관성 높음 (Python 3개, Node.js 2개)
- 코드 재사용 가능 (같은 언어 레이어 간)
- 빠른 개발 속도
- 1인 개발 환경에 적합

---

## 레이어별 상세

### 1. 데이터 수집 레이어 (Python)

**선택 이유**
- 웹 스크래핑 라이브러리 풍부 (BeautifulSoup, Scrapy)
- RSS 파싱 간단 (feedparser)
- 비동기 HTTP 클라이언트 (httpx, aiohttp)

**주요 라이브러리**
- `feedparser` - RSS 파싱
- `httpx` - HTTP 클라이언트
- `beautifulsoup4` - HTML 파싱
- `redis` - Redis Streams
- `oracledb` - Oracle 연결

---

### 2. 분석 레이어 (Python)

**선택 이유**
- LLM API 호출 간단 (requests, httpx)
- JSON 처리 네이티브
- 수집 레이어와 코드 재사용 가능
- 빠른 프로토타이핑

**주요 라이브러리**
- `httpx` - Gemini API 호출
- `redis` - Redis Streams
- `oracledb` - Oracle 연결

**고려사항**
- LLM 응답 JSON 검증은 Pydantic 또는 dataclass로 보완
- 타입 힌트 적극 활용으로 런타임 에러 최소화

---

### 3. 중복 제거 레이어 (Python)

**선택 이유**
- **SentenceTransformers 네이티브 지원** (핵심)
- 임베딩/ML 생태계 최강
- Qdrant Python 클라이언트 공식 지원
- 다른 언어 선택 시 별도 임베딩 서비스 필요 (복잡도 증가)

**주요 라이브러리**
- `sentence-transformers` - 임베딩 생성
- `qdrant-client` - 벡터 DB
- `redis` - Redis Streams
- `oracledb` - Oracle 연결

**임베딩 모델**
- all-MiniLM-L12-v2 또는 all-MiniLM-L6-v2
- 차원: 384

---

### 4. 피드 생성 레이어 (Node.js)

**선택 이유**
- API 레이어와 동일 언어로 타입 정의 공유 가능
- TypeScript 사용 시 피드 구조 타입 안전성 확보
- 비동기 I/O에 최적화

**주요 라이브러리**
- `ioredis` - Redis Streams
- `oracledb` - Oracle 연결

**장점**
- API와 피드 데이터 타입 공유
- Node.js 생태계 일관성

---

### 5. API 레이어 (Node.js)

**선택 이유**
- 비동기 I/O에 최적화
- Express/Fastify 생태계 활용

**주요 라이브러리**
- `express` 또는 `fastify` - HTTP 서버
- `ioredis` - Redis Streams
- `oracledb` - Oracle 연결

---

## 기술 스택 요약

### Python 레이어 (수집, 분석, 중복 제거)

| 용도 | 라이브러리 |
|------|-----------|
| HTTP 클라이언트 | httpx |
| Redis | redis |
| Oracle | oracledb |
| JSON 검증 | pydantic |
| 로깅 | structlog |

### Node.js 레이어 (피드 생성, API)

| 용도 | 라이브러리 |
|------|-----------|
| HTTP 서버 | express 또는 fastify |
| Redis | ioredis |
| Oracle | oracledb |

---

## 배포 구성

### 컨테이너 구성

| 컨테이너 | 언어 | 레이어 |
|----------|------|--------|
| collector | Python | 데이터 수집 |
| analyzer | Python | 분석 |
| deduplicator | Python | 중복 제거 |
| feed-generator | Node.js | 피드 생성 |
| api-server | Node.js | API |

**총 애플리케이션 컨테이너**: 5개

### 인프라 컨테이너

| 컨테이너 | 용도 |
|----------|------|
| redis | Key-Value Store + Message Queue |
| qdrant | Vector DB |

**총 인프라 컨테이너**: 2개

---

## 데이터 흐름

```
[데이터 소스]
     ↓
[수집 레이어 - Python]
     ↓ Redis Streams (queue:collection)
[분석 레이어 - Python]
     ↓ Redis Streams (queue:analysis)
[중복 제거 레이어 - Python]
     ↓ Redis Streams (queue:deduplication)
[피드 생성 레이어 - Node.js]
     ↓ Redis Streams (queue:feed)
[API 레이어 - Node.js]
     ↓
[클라이언트]
```