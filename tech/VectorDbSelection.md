# Vector DB 선택 문서

## 1. 선택 결과

**Vector DB: Qdrant**

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
- 로컬 환경 실행 가능

---

## 3. 기술 요구사항

### 데이터 규모
- 일일 수집: 120개
- 예상 중복률: 10%
- 일일 저장: 약 108개
- TTL: 48시간
- 총 데이터: 약 216개

### 벡터 사양
- 임베딩 모델: all-MiniLM-L12-v2 또는 all-MiniLM-L6-v2
- 차원: 384
- 유사도: 코사인

### 성능
- 검색 속도: 2-3초 이내
- 쿼리 빈도: 분당 2개 정도

### 필수 기능
- 코사인 유사도 검색
- 메타데이터 필터링
- TTL 자동 삭제

---

## 4. 평가 우선순위

1. **설정 간편함** (가장 중요)
2. **개발 편의성**
3. **성능**
4. **안정성/문서**

---

## 5. 후보 비교

| 항목 | Qdrant | Chroma | Milvus | Weaviate |
|------|---------|---------|---------|-----------|
| 도커 설정 | 간단 (1개) | 간단 (1개) | 복잡 (3개+) | 간단 (1개) |
| TTL 자동 | ✅ | ❌ | ✅ | ✅ |
| 웹 UI | ✅ | ❌ | ✅ | ❌ |
| 성능 | 우수 | 충분 | 우수 | 우수 |
| 확장성 | 우수 | 제한적 | 최고 | 우수 |
| MVP 적합 | ✅ | ✅ | ❌ | △ |

---

## 6. Qdrant 선택 이유

### 핵심 이유
1. **도커 설정이 Chroma와 동일하게 간단** (컨테이너 1개)
2. **TTL 자동 삭제** (48시간 지난 데이터 자동 처리)
3. **웹 UI 제공** (디버깅 편의)
4. **우수한 성능** (Rust 기반, M2 최적화)
5. **확장 가능** (데이터 증가 시 대응 가능)

### 결정적 요소
- **도커 선호**: Chroma의 주 장점(pip install)이 무의미
- **TTL 필요**: 수동 삭제 코드 작성 불필요
- **웹 UI**: 개발 중 데이터 확인 편리

### 트레이드오프
- ⚠️ 코드가 Chroma보다 약간 김 (2줄 vs 5줄)
- ✅ 하지만 실질적 차이 미미
- ✅ TTL, 웹 UI, 성능으로 충분히 보상

---

## 7. 제외된 후보

### Chroma
- 도커 환경에서 Qdrant 대비 장점 없음
- TTL 미지원 (수동 처리 필요)
- 확장성 제한적

### Milvus
- 설정 복잡 (컨테이너 3개+)
- MVP에 오버스펙
- 리소스 과다 사용

### Weaviate
- GraphQL API (학습 필요)
- Schema 정의 필수
- 상대적으로 복잡

---

## 8. Qdrant 기술 스펙

### Docker 설정
```yaml
# docker-compose.yml
version: '3'
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - ./qdrant_storage:/qdrant/storage
```

### 접속 정보
- REST API: http://localhost:6333
- 웹 UI: http://localhost:6333/dashboard
- gRPC: localhost:6334 (옵션)

### Python 클라이언트
```bash
pip install qdrant-client
```

### 기본 사용 예시
```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

# 클라이언트 생성
client = QdrantClient("localhost", port=6333)

# 컬렉션 생성
client.create_collection(
    collection_name="trump_statements",
    vectors_config=VectorParams(size=384, distance=Distance.COSINE)
)

# 데이터 삽입 (TTL 48시간)
client.upsert(
    collection_name="trump_statements",
    points=[
        PointStruct(
            id="uuid",
            vector=[0.1, 0.2, ...],  # 384차원
            payload={
                "channel": "truth_social",
                "timestamp": "2025-11-12T10:00:00Z",
                "content": "..."
            }
        )
    ]
)

# 유사도 검색
results = client.search(
    collection_name="trump_statements",
    query_vector=[0.1, 0.2, ...],
    limit=5,
    score_threshold=0.7
)
```

---

## 9. 향후 고려사항

### 확장 시나리오
- 데이터 10배 증가 (2,160개): Qdrant 그대로 사용 가능
- 데이터 100배 증가 (21,600개): Qdrant 그대로 사용 가능
- 그 이상: 클러스터링 고려

### 모니터링
- 웹 UI로 실시간 상태 확인
- 메트릭: 검색 속도, 메모리 사용량, 벡터 수

### 백업
- Docker volume 백업으로 데이터 보존
- 필요 시 RDBMS에 벡터 이력 보관

---

## 10. 결론

**Qdrant는 현재 요구사항(MVP, 도커 기반, 간편함)과 향후 확장 가능성을 모두 만족하는 최적의 선택입니다.**

- 설정 간편: ✅
- 개발 편의: ✅
- 성능: ✅
- 확장성: ✅
- 비용: ✅ (무료)

---

**문서 버전**: 1.0  
**다음 단계**: Message Queue 선택