# Redis Stream 정의 문서

## 1. 개요

본 문서는 Trump Scan 서비스의 레이어 간 비동기 통신에 사용되는 Redis Stream들의 명세를 정의합니다. 각 Stream의 이름, Producer/Consumer, 메시지 스키마를 기술합니다.

### 1.1 Stream 네이밍 규칙

**패턴**: `trump-scan:{layer-name}:{data-type}`

**구성 요소**:
- `trump-scan`: 프로젝트 네임스페이스
- `{layer-name}`: 레이어 이름 (kebab-case)
- `{data-type}`: 데이터 타입 (kebab-case)

**예시**:
- `trump-scan:data-collection:raw-data`
- `trump-scan:analysis:analysis-result`

### 1.2 DLQ (Dead Letter Queue) 네이밍 규칙

각 Stream마다 DLQ Stream을 구성합니다.

**패턴**: `{stream-name}:dlq`

**예시**:
- `trump-scan:data-collection:raw-data:dlq`
- `trump-scan:analysis:analysis-result:dlq`

**용도**:
- 최대 재시도 횟수 초과 시 메시지 이동
- 파싱 불가능하거나 잘못된 메시지 격리
- 수동 검토 및 재처리

---

## 2. Stream 목록

### 2.1 데이터 수집 → 분석

**Stream 이름**: `trump-scan:data-collection:raw-data`

**Producer**: Data Collection Layer
**Consumer**: Analysis Layer
**Consumer Group**: `analysis-workers`

**DLQ Stream**: `trump-scan:data-collection:raw-data:dlq`

**용도**: 수집된 원본 데이터를 분석 레이어로 전달

**메시지 스키마**:
```json
{
  "id": 700,
  "content": "Why aren't they under arrest for sedition......thrown out of their offices...ENOUGH IS ENOUGH...",
  "link": "https://trumpstruth.org/statuses/33909",
  "published_at": "2025-11-19T18:07:25Z",
  "channel": "truth_social"
}
```

**필드 설명**:
- `id`: 원본 데이터의 고유 ID (RDBMS Primary Key, integer)
- `content`: 수집된 발언 원문
- `link`: 원본 링크
- `published_at`: 발언 발생 시간 (ISO 8601)
- `channel`: 데이터 수집 채널 (`truth_social`, `twitter`, `news`, `press_release`)

---

### 2.2 분석 → 중복 제거

**Stream 이름**: `trump-scan:analysis:analysis-result`

**Producer**: Analysis Layer
**Consumer**: Deduplication Layer
**Consumer Group**: `dedup-workers`

**DLQ Stream**: `trump-scan:analysis:analysis-result:dlq`

**용도**: 분석 완료된 데이터를 중복 제거 레이어로 전달

**메시지 스키마**: 레이어 구현 후 작성

---

### 2.3 중복 제거 → 피드 생성

**Stream 이름**: `trump-scan:deduplication:unique-content`

**Producer**: Deduplication Layer
**Consumer**: Feed Generation Layer
**Consumer Group**: `feed-workers`

**DLQ Stream**: `trump-scan:deduplication:unique-content:dlq`

**용도**: 중복 제거를 통과한 고유 콘텐츠를 피드 생성 레이어로 전달

**메시지 스키마**: 레이어 구현 후 작성

---

### 2.4 피드 생성 → API

**Stream 이름**: `trump-scan:feed-generation:new-feed`

**Producer**: Feed Generation Layer
**Consumer**: API Layer
**Consumer Group**: `api-notifiers`

**DLQ Stream**: `trump-scan:feed-generation:new-feed:dlq`

**용도**: 새로운 피드 항목 생성 알림을 API 레이어로 전달 (실시간 전송용)

**메시지 스키마**: 레이어 구현 후 작성

---

## 3. Stream 구성 요약

| From Layer | To Layer | Stream Name | Consumer Group |
|-----------|----------|-------------|----------------|
| 수집 | 분석 | `trump-scan:data-collection:raw-data` | `analysis-workers` |
| 분석 | 중복제거 | `trump-scan:analysis:analysis-result` | `dedup-workers` |
| 중복제거 | 피드생성 | `trump-scan:deduplication:unique-content` | `feed-workers` |
| 피드생성 | API | `trump-scan:feed-generation:new-feed` | `api-notifiers` |

---

## 4. Consumer Group 전략

### 4.1 기본 원칙

- 각 레이어는 하나의 Consumer Group 사용
- Consumer Group은 레이어 역할을 나타내는 이름 사용 (예: `analysis-workers`, `dedup-workers`)
- 동일 Consumer Group 내에서 여러 Consumer 실행 가능 (수평 확장)

### 4.2 Consumer 명명

**패턴**: `{hostname}-{process-id}` 또는 `worker-{instance-number}`

**예시**:
- `worker-1`
- `worker-2`
- `server-a-12345`

---

## 5. 메시지 처리 보장

### 5.1 At-Least-Once Delivery

- Redis Streams의 Consumer Group 사용으로 At-Least-Once 보장
- 각 레이어는 멱등성(Idempotency) 고려 필요

### 5.2 ACK 전략

**처리 완료 후 ACK**:
1. 메시지 수신
2. 데이터 저장 완료
3. 다음 레이어로 메시지 발행 완료
4. ACK 전송

**실패 시**:
- ACK 하지 않음
- 재시도 (최대 3회)
- 최종 실패 시 DLQ로 이동

---

## 6. 모니터링 메트릭

### 6.1 Stream별 메트릭

각 Stream에 대해 다음 메트릭을 추적합니다:

- **Queue Length**: 현재 대기 중인 메시지 수
- **Processing Rate**: 초당 처리된 메시지 수
- **Lag**: Consumer의 처리 지연 시간
- **DLQ Size**: DLQ에 쌓인 메시지 수

### 6.2 조회 명령

**Stream 길이 확인**:
```bash
redis-cli XLEN trump-scan:data-collection:raw-data
```

**Consumer Group 정보**:
```bash
redis-cli XINFO GROUPS trump-scan:data-collection:raw-data
```

**Pending 메시지 확인**:
```bash
redis-cli XPENDING trump-scan:data-collection:raw-data analysis-workers
```

---

## 7. 관련 문서

- [Message Queue 선택 문서](./MessageQueueSelection.md): Redis Streams 선택 배경 및 기술 스펙
- [데이터 처리 파이프라인 설계](../0.DataProcessingPipelineDesign.md): 전체 아키텍처 및 메시지 계약
- [각 레이어별 설계 문서](../): 각 레이어의 상세 구현 설계
