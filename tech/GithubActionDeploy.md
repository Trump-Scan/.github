# GitHub Actions 배포 전략

## 문서 개요

본 문서는 Trump Scan 서비스의 **API 레이어를 제외한 4개 레이어**(데이터 수집, 분석, 중복 제거+피드 생성)를 GitHub Actions로 실행하는 방안을 정리합니다.

**상태**: 검토 완료, 구현 대기  

---

## 1. 배경 및 동기

### 1.1 현재 상황

- **OCI 프리티어 VM**: AMD EPYC 2 vCPU, 1GB RAM
- **현재 용도**: 개인 프로젝트 웹사이트 (프론트/백엔드/Nginx)
- **여유 리소스**: RAM 522MB
- **제약사항**: ARM A1 (4 OCPU, 24GB) 리전별 수량 제한으로 사실상 구할 수 없음

### 1.2 Trump Scan 리소스 요구사항

각 레이어별 예상 리소스 (실제 코드 기반 분석):

| 레이어 | CPU | RAM | 비고 |
|--------|-----|-----|------|
| 데이터 수집 | 0.1 | 170MB | RSS 파싱, HTTP I/O (feedparser, httpx 등) |
| 분석 | 0.1 | 150MB | LLM API I/O 대기 (google-genai SDK) |
| 중복 제거 + 피드 생성 | 0.4-0.6 | 800MB | **SentenceTransformers 모델 로드** (CPU intensive) |
| **합계** | **~0.7** | **~1.1GB** | |

**핵심 발견:**
- 데이터 수집/분석 레이어는 I/O bound로 매우 가벼움 (각 150-170MB)
- 중복 제거 레이어가 가장 무거움 (SentenceTransformers 모델 500-600MB + 런타임)
- 임베딩 생성이 유일한 CPU intensive 작업 (0.4-0.6 core)

**판단**: 현재 VM에 배포 불가능 (RAM 0.6GB 부족)

### 1.3 선택 동기

- **로컬 PC 24시간 실행 회피**: 휴먼 리소스 절감
- **인프라 비용 최소화**: OCI VM 추가 불가, 다른 무료 서비스도 RAM 부족
- **실현 가능성 검토**: GitHub Actions의 적합성 평가

---

## 2. GitHub Actions 적합성 분석

### 2.1 기술적 실현 가능성

#### ✅ 가능한 이유

1. **배치 작업 특성**: 수집→분석→중복제거→피드생성은 모두 **이벤트 기반 배치 처리**
2. **충분한 리소스**: GitHub runner (7GB RAM, 2 core)
3. **외부 인프라 연결**: Redis (OCI), Oracle DB (OCI)와 네트워크 통신 가능
4. **Secrets 관리**: 민감 정보를 GitHub Secrets로 관리

### 2.2 비용 분석

#### ⚠️ 중요: Public vs Private Repository

GitHub Actions 요금은 **repository 공개 여부에 따라 완전히 다릅니다**:

| Repository 타입 | 무료 제공 | 초과 요금 |
|----------------|----------|-----------|
| **Public** | **무제한 무료** | 없음 |
| **Private** (Free tier) | 2,000분/월 | $0.008/분 |
| **Private** (Pro, $4/월) | 3,000분/월 | $0.008/분 |

#### 예상 사용량 계산

```
레이어별 실행 시간 (4-layer 아키텍처):
- 데이터 수집: 2분
- 분석: 2분
- 중복 제거 + 피드 생성: 2분
합계: 6분/회

실행 빈도 (10분 간격):
- 시간당: 6회
- 일일: 144회 × 6분 = 864분
- 월간: 864분 × 30일 = 25,920분
```

**참고**: 피드 생성은 중복 제거 레이어에 통합되어 별도 시간 불필요

#### 비용 산출

##### Case 1: Public Repository

```
월 필요량: 25,920분
비용: $0 (무제한 무료)
```

**결론**: ✅ **Public repo면 비용 문제 없음!**

##### Case 2: Private Repository

```
월 필요량: 25,920분
Free tier: 2,000분
초과분: 23,920분

초과 요금: 23,920분 × $0.008 = $191.36/월
```

**결론**: ❌ **Private repo면 월 $191 비용 발생**

### 2.3 비용 절감 방안

#### 옵션 1: 실행 빈도 감소

| 간격 | 시간당 | 일일 | 월간 | 초과분 | 비용 |
|------|-------|------|------|--------|------|
| 10분 | 6회 | 864분 | 25,920분 | 23,920분 | **$191/월** |
| 15분 | 4회 | 576분 | 17,280분 | 15,280분 | **$122/월** |
| 20분 | 3회 | 432분 | 12,960분 | 10,960분 | **$88/월** |
| 30분 | 2회 | 288분 | 8,640분 | 6,640분 | **$53/월** |
| 60분 | 1회 | 144분 | 4,320분 | 2,320분 | **$19/월** |

**트레이드오프**: 실행 빈도 감소 = 실시간성 저하

#### 옵션 2: 레이어 통합

4개 workflow를 1개로 통합 (순차 실행):
- 월 사용량: 25,920분 / 4 = 6,480분
- 초과분: 4,480분 × $0.008 = **$36/월**

**문제점**: 한 레이어 실패 시 전체 중단

#### 옵션 3: 선택적 적용

일부 레이어만 GitHub Actions 사용:
- 예) 피드 생성만 GitHub Actions
- 나머지는 로컬 실행

**문제점**: 하이브리드 관리 복잡도

### 2.4 최종 비용 비교

| 방식 | 월 비용 | 실시간성 | 관리 복잡도 | 안정성 |
|------|---------|----------|------------|--------|
| 로컬 PC | 전기세 (~$5) | 5분 | 낮음 | 중 |
| GitHub Actions (10분) | **$191** | 15분 | 높음 | 중 |
| GitHub Actions (60분) | **$19** | 60분+ | 높음 | 중 |
| OCI VM 추가 | $0 | 5분 | 낮음 | 높음 |

**판단**: GitHub Actions는 **비용 측면에서 실용적이지 않음**

---

## 3. 기술적 구현 방안 (참고용)

> **주의**: 비용 문제로 권장하지 않지만, 기술적으로는 다음과 같이 구현 가능합니다.

### 3.1 전체 구조

```
GitHub Actions (각 레이어 독립 실행)
  
Data Collection Repo (10분마다)
  → [Redis Streams] → raw-data
  
Analysis Repo (10분마다)
  ← [Redis Streams] ← raw-data
  → [Redis Streams] → analysis-result
  
Deduplication Repo (10분마다)
  ← [Redis Streams] ← analysis-result
  → [Oracle DB] → feed_items
```

**핵심 특징:**
- 각 레이어는 **독립적으로 10분마다 실행**
- Redis Streams가 자연스럽게 레이어 연결
- 한 레이어 실패해도 다른 레이어는 계속 작동
- 메시지 큐가 버퍼 역할 (처리 시차 흡수)

**실행 흐름 예시:**
```
10:00 - Collection 실행 (데이터 수집 → 메시지 발행)
        Analysis 실행 (메시지 소비 → 분석 → 메시지 발행)
        Dedup 실행 (메시지 소비 → 처리)

10:10 - Collection 실행 (데이터 수집 → 메시지 발행)
        Analysis 실행 (메시지 소비 → 분석 → 메시지 발행)
        Dedup 실행 (메시지 소비 → 처리)

10:20 - 모든 레이어가 다시 동시 실행...
```

**핵심**: 각 레이어는 **동시에 10분마다** 실행되며, Redis Streams 큐에서 메시지를 가져가 처리

### 3.2 코드 수정

#### main.py (데이터 수집)

```python
if __name__ == "__main__":
    import sys
    
    if "--once" in sys.argv:
        # GitHub Actions용: 1회 실행 후 종료
        orchestrator.run()
    else:
        # 로컬 실행용: 스케줄러로 무한 루프
        orchestrator.start()
```

#### worker.py (분석 / 중복제거+피드생성)

```python
def run_once(self):
    """1회 처리 후 종료 (GitHub Actions용)"""
    raw_data = self._message_subscriber.receive()
    
    if raw_data is None:
        logger.info("처리할 메시지 없음")
        return
    
    # 처리 로직
    # ...
```

**참고**: 중복 제거 레이어의 worker는 피드 생성 로직까지 포함

### 3.3 Workflow 설정

#### 동시 실행 방지 (필수)

모든 workflow에 다음 설정 필수:

```yaml
concurrency:
  group: ${{ github.workflow }}
  cancel-in-progress: false
```

**왜 필요한가?**
- Analysis 레이어는 LLM 호출로 10분 이상 걸릴 수 있음
- 이전 실행이 끝나기 전에 다음 스케줄 실행이 시작될 수 있음
- 동시 실행 시 Redis/DB 접근 충돌, 중복 처리 발생

**설정 의미:**
- `group`: workflow별로 동시 실행 제어 그룹
- `cancel-in-progress: false`: 실행 중이면 새 실행 **스킵** (취소 안 함)

---

## 4. 장단점 요약

### ❌ 단점 (Private repo인 경우)

1. **비용**: 월 $19-191 (실행 빈도에 따라)
2. **스케줄 지연**: 실제 5-10분 추가 지연
3. **관리 복잡도**: 3개 workflow, secrets 관리
4. **모니터링 제약**: 제한적인 로깅
5. **안정성**: GitHub 의존성

### ✅ 장점 (Public repo인 경우)

1. **완전 무료**: 무제한 사용
2. **리소스 충분**: 7GB RAM, 2 core
3. **관리 자동화**: Git 기반 배포
4. **로컬 PC 불필요**: 전기세 절감

---

## 5. 결론 및 권장사항

### 5.1 최종 판단

#### ✅ **Public Repository: 매우 적합!**

1. **기술적 실현 가능성**: ✅ 가능
2. **리소스 충족**: ✅ 충분
3. **비용**: ✅ **완전 무료!**
4. **실시간성**: ⚠️ 15분+ 지연 (허용 가능)

**결론**: Public repo로 운영한다면 GitHub Actions가 **최선의 선택**

#### ❌ **Private Repository: 부적합**

1. **기술적 실현 가능성**: ✅ 가능
2. **리소스 충족**: ✅ 충분
3. **비용**: ❌ **월 $19-191 발생**
4. **실시간성**: ⚠️ 15분+ 지연

**결론**: Private repo라면 로컬 PC 실행 권장

### 5.2 권장 방안

#### Public Repository인 경우

**Option 1: GitHub Actions** (강력 권장 ⭐)
- 비용: **$0** (완전 무료)
- 실시간성: 10-15분
- 장점: 관리 편리, PC 불필요
- 단점: 코드 공개, 약간의 지연

**Option 2: 로컬 PC 실행**
- 비용: 전기세만 (~$5/월)
- 실시간성: 5분
- 장점: 완전한 제어
- 단점: PC 24시간 가동

#### Private Repository인 경우

**Option 1: 로컬 PC 실행** (권장)
- 비용: 전기세만 (~$5/월)
- 실시간성: 5분
- 단점: PC 24시간 가동

**Option 2: GitHub Actions (실행 빈도 감소)**
- 60분 간격 실행
- 비용: $19/월
- 단점: 실시간성 저하

#### 중기 (사용자 확보 후)

1. **OCI ARM A1 슬롯 재시도**
    - 자동화 스크립트 활용
    - 리전 변경 시도

2. **OCI PAYG 전환**
    - Always Free 혜택 유지
    - 리소스 우선순위 상승

#### 장기 (프로덕션)

- OCI VM 확장
- AWS/GCP 등 유료 tier

### 5.3 GitHub Actions 활용 시나리오

**강력 권장하는 경우**:

1. ✅ **Public repository로 운영** (완전 무료!)
2. ✅ 10-15분 지연 허용 가능
3. ✅ 관리 편의성 중시

**고려할 만한 경우** (Private repo):

1. ✅ 실행 빈도가 낮은 경우 (60분 이상 간격)
2. ✅ 일부 레이어만 적용
3. ✅ 이미 GitHub Pro 구독 중

**부적합한 경우**:

1. ❌ Private repo + 실시간성 중요
2. ❌ Private repo + 비용 최소화 목표

---

## 6. 참고 자료

### 6.1 공식 문서

- **GitHub Actions Pricing**: https://docs.github.com/en/billing/managing-billing-for-github-actions
- **Runner 스펙**: https://docs.github.com/en/actions/using-github-hosted-runners

### 6.2 관련 이슈

- OCI ARM A1 부족 문제: https://github.com/hitrov/oci-arm-host-capacity
- GitHub Actions 스케줄 지연: https://github.com/orgs/community/discussions/26726