# 임베딩 전략 리서치 결과

> 작성일: 2026-02-07
> 환경: Apple M1 Pro 16GB, Node.js v24.13.0, macOS 26.2

---

## 1. 배경

DESIGN.md v8에서 `claude --print`로 384차원 벡터를 생성하려 했으나, Claude는 임베딩 모델이 아니라 비현실적.
Anthropic은 자체 임베딩 API를 제공하지 않으며, Voyage AI를 공식 파트너로 권장.

### 용도
- 에러 KB 유사도 검색 (error-kb.mjs)
- 스킬 매칭 (skill-matcher.mjs)
- 한국어 ↔ 영어 교차 언어 매칭 필요

---

## 2. 테스트한 모델 (5종)

### 2.1 Transformers.js — multilingual-e5-small

| 항목 | 값 |
|------|-----|
| 패키지 | `@xenova/transformers` (v2.17.2) |
| 모델 | `Xenova/multilingual-e5-small` |
| 차원 | 384 |
| 모델 캐시 | 129 MB |
| 메모리 사용 | 689 MB |
| 첫 로드 | 3.6s |
| 캐시 재로드 | 0.58s |
| 단일 임베딩 | 4~8ms |
| 배치 10건 | 27ms (2.7ms/건) |
| 입력 프리픽스 | `query: ` / `passage: ` 필요 |
| 비용 | 무료 |
| 오프라인 | 가능 (첫 실행 시 모델 다운로드) |

#### 에러 KB 시뮬레이션 결과

KB에 5개 에러를 등록한 후 8가지 쿼리로 테스트:

| 쿼리 | 기대 매칭 | Top-1 | dist | 결과 |
|------|----------|-------|------|------|
| ENOENT (다른 경로) | ENOENT | ENOENT | 0.0728 | ✅ |
| "파일을 찾을 수 없습니다" (KR) | ENOENT | Module not found | 0.1800 | ❌ (한국어 약점) |
| "permission denied" (EN) | EACCES | EACCES | 0.1593 | ✅ |
| "Cannot read property 'length'" | TypeError | TypeError | 0.1239 | ✅ |
| "react 모듈이 없습니다" (KR) | Module not found | Module not found | 0.1198 | ✅ |
| "데이터베이스 연결이 안됩니다" (KR) | ECONNREFUSED | Module not found | 0.1879 | ❌ |
| "배포 스크립트를 작성해줘" | 매칭 없음 | - | 0.1925 | ❌ (오탐) |
| "커밋 메시지 작성해줘" | 매칭 없음 | - | 0.1928 | ✅ |

**Top-1 정확도**: 5/6 (한국어 순수 텍스트 2건 실패)

#### 임계값별 정확도

| 임계값 | Precision | Recall | F1 | 비고 |
|--------|-----------|--------|-----|------|
| 0.15 | 1.00 | 0.50 | 0.67 | 정확하지만 놓침 많음 |
| **0.17** | **1.00** | **0.67** | **0.80** | **최적 (오탐 0)** |
| 0.18 | 0.80 | 0.80 | 0.80 | 오탐 1건 시작 |
| 0.19 | 0.67 | 1.00 | 0.80 | 오탐 증가 |
| 0.20+ | 0.50 | 1.00 | 0.67 | 오탐 급증 |

#### 유사도 분포

| 비교 | distance |
|------|----------|
| ENOENT (EN) ↔ ENOENT (KR) | 0.0581 |
| ENOENT (EN) ↔ EACCES (EN) | 0.1331 |
| ENOENT (EN) ↔ TypeError (EN) | 0.1428 |
| ENOENT (EN) ↔ "관련 없는 문장" | 0.1744 |
| ENOENT (EN) ↔ "전혀 관련 없는" | 0.2195 |

**핵심 관찰**: 분포가 매우 압축됨 (distance 0.05~0.22 범위). 기존 DESIGN.md의 threshold 0.3으로는 모든 것이 매칭됨.

#### 장단점
- ✅ 완전 무료, 오프라인, npm install만으로 설치
- ✅ 임베딩 생성 매우 빠름 (4~8ms)
- ✅ 영어 에러 코드 포함 시 교차 언어 매칭 양호
- ✅ `query:`/`passage:` 프리픽스로 검색 최적화
- ❌ 순수 한국어 → 영어 매칭 약함
- ❌ 유사도 분포 압축으로 임계값 설정 까다로움 (유효 범위 0.02)

---

### 2.2 ⭐ Transformers.js — paraphrase-multilingual-MiniLM-L12-v2 (추천)

| 항목 | 값 |
|------|-----|
| 패키지 | `@xenova/transformers` (v2.17.2) |
| 모델 | `Xenova/paraphrase-multilingual-MiniLM-L12-v2` |
| 차원 | 384 |
| 메모리 사용 | 717 MB |
| 첫 로드 | 3.72s |
| 단일 임베딩 | 2.4ms |
| 입력 프리픽스 | 불필요 |
| 비용 | 무료 |
| 오프라인 | 가능 |

#### 에러 KB 시뮬레이션 결과

| 쿼리 | 기대 매칭 | Top-1 | dist | gap | 결과 |
|------|----------|-------|------|-----|------|
| ENOENT (다른 경로) | ENOENT | ENOENT | 0.1243 | 0.5502 | ✅ |
| "파일을 찾을 수 없습니다" (KR) | ENOENT | ENOENT | 0.6666 | 0.1237 | ✅ |
| "permission denied" (EN) | EACCES | EACCES | 0.4470 | 0.1758 | ✅ |
| "Cannot read property 'length'" | TypeError | TypeError | 0.4703 | 0.3845 | ✅ |
| "react 모듈이 없습니다" (KR) | Module not found | Module not found | 0.4627 | 0.2622 | ✅ |
| "데이터베이스 연결이 안됩니다" (KR) | ECONNREFUSED | ECONNREFUSED | 0.7343 | 0.0837 | ✅ |
| "배포 스크립트를 작성해줘" | 매칭 없음 | - | 0.8773 | 0.0288 | ✅ |
| "커밋 메시지 작성해줘" | 매칭 없음 | - | 0.9179 | 0.0032 | ✅ |

**Top-1 정확도**: 6/6 (100%) — 한국어 순수 텍스트 포함 모든 쿼리 정확

#### 임계값별 정확도

| 임계값 | Precision | Recall | F1 | 비고 |
|--------|-----------|--------|-----|------|
| 0.48 | 1.00 | 0.67 | 0.80 | 영어 + 에러코드 포함 매칭 |
| 0.68 | 1.00 | 0.83 | 0.91 | + 한국어 연결 에러 매칭 |
| **0.76** | **1.00** | **1.00** | **1.00** | **완벽 (오탐 0, 누락 0)** |
| 0.78 | 1.00 | 1.00 | 1.00 | 안전 마진 확보 |

#### 유사도 분포

| 비교 | distance |
|------|----------|
| ENOENT (EN) ↔ ENOENT (KR) | 0.4550 |
| ENOENT (EN) ↔ EACCES (EN) | 0.6463 |
| ENOENT (EN) ↔ TypeError (EN) | 0.7691 |
| ENOENT (EN) ↔ "관련 없는 문장" | 0.8690 |
| ENOENT (EN) ↔ "전혀 관련 없는" | 0.9338 |

**핵심 관찰**: 분포가 매우 넓음 (distance 0.12~0.93). 매칭/비매칭 경계가 명확함 (0.7343 vs 0.8773 → gap 0.143).

#### 거리 구간 분석

```
매칭 있음 (에러 쿼리):   0.12 ─────────────── 0.73
매칭 없음 (무관 쿼리):                               0.88 ── 0.92
                                              ↑
                                         안전 구간 0.143
```

임계값을 0.76으로 설정하면 매칭/비매칭 양쪽에서 충분한 마진 확보.

#### 스킬 매칭 테스트

| 순위 | 쿼리: "도커 이미지를 빌드해줘" | dist |
|------|-------------------------------|------|
| 1 | Git 브랜치 정리 및 태그 관리 | 0.7009 |
| 2 | Build Docker images and push to registry | 0.9398 |
| 3 | Format markdown documentation files | 0.9782 |

**스킬 매칭 약점**: 한국어 → 영어 교차 언어 의미 검색에 약함. "도커 빌드"가 "Build Docker"보다 "Git 브랜치 정리"에 더 가까움. → **스킬 설명에 한국어 포함 필요** (예: "Docker 이미지 빌드 / Build Docker images")

#### 장단점
- ✅ **에러 KB F1 = 1.00** (테스트한 모든 모델 중 최고)
- ✅ 분포가 넓어 임계값 설정 용이 (유효 범위 0.14)
- ✅ 한국어 순수 텍스트 매칭 성공 ("파일을 찾을 수 없습니다" → ENOENT)
- ✅ 프리픽스 불필요 (구현 단순)
- ✅ 빠름 (2.4ms/건)
- ❌ 한국어 → 영어 의미 검색 약함 (스킬 매칭 시 한국어 설명 필요)

---

### 2.3 Transformers.js — bge-small-en-v1.5

| 항목 | 값 |
|------|-----|
| 모델 | `Xenova/bge-small-en-v1.5` |
| 차원 | 384 |
| 메모리 사용 | 1158 MB |
| 첫 로드 | 1.22s |
| 단일 임베딩 | 3.7ms |
| 다국어 | ❌ 영어 전용 |
| 입력 프리픽스 | 쿼리: `Represent this sentence...` |

#### 에러 KB 시뮬레이션 결과

| 쿼리 | Top-1 | dist | 결과 |
|------|-------|------|------|
| ENOENT (다른 경로) | ENOENT | 0.1441 | ✅ |
| "파일을 찾을 수 없습니다" (KR) | ENOENT | 0.6094 | ✅ |
| "permission denied" (EN) | EACCES | 0.4268 | ✅ |
| "Cannot read property 'length'" | TypeError | 0.3610 | ✅ |
| "react 모듈이 없습니다" (KR) | Module not found | 0.4391 | ✅ |
| "데이터베이스 연결이 안됩니다" (KR) | TypeError❌ | 0.5459 | ❌ |

**Top-1 정확도**: 5/6 (한국어 연결 에러 오매칭)

#### 임계값별 정확도

최적: t=0.50 → F1=0.80 (P=1.00, R=0.67)

**결론**: 영어 전용 모델이므로 한국어 매칭 불안정. 메모리도 1158MB로 높음. 부적합.

---

### 2.4 Transformers.js — gte-small

| 항목 | 값 |
|------|-----|
| 모델 | `Xenova/gte-small` |
| 차원 | 384 |
| 메모리 사용 | 1159 MB |
| 첫 로드 | 2.41s |
| 단일 임베딩 | 2.3ms |
| 다국어 | 제한적 |
| 입력 프리픽스 | 불필요 |

#### 에러 KB 시뮬레이션 결과

| 쿼리 | Top-1 | dist | 결과 |
|------|-------|------|------|
| ENOENT (다른 경로) | ENOENT | 0.0648 | ✅ |
| "파일을 찾을 수 없습니다" (KR) | ECONNREFUSED❌ | 0.2140 | ❌ |
| "permission denied" (EN) | EACCES | 0.1654 | ✅ |
| "Cannot read property 'length'" | TypeError | 0.1613 | ✅ |
| "react 모듈이 없습니다" (KR) | Module not found | 0.1838 | ✅ |
| "데이터베이스 연결이 안됩니다" (KR) | ECONNREFUSED | 0.2256 | ✅ |

**Top-1 정확도**: 5/6 (한국어 "파일을 찾을 수 없습니다" 오매칭)

#### 유사도 분포

distance 0.06~0.26 — e5-small과 유사하게 압축됨. 최적 t=0.20 (F1=0.80).

**결론**: e5-small과 유사한 성능이지만 메모리가 1159MB로 2배. 이점 없음.

---

### 2.5 Transformers.js — all-MiniLM-L6-v2

| 항목 | 값 |
|------|-----|
| 모델 | `Xenova/all-MiniLM-L6-v2` |
| 차원 | 384 |
| 다국어 | ❌ 영어 전용 |

#### 유사도 분포

| 비교 | distance |
|------|----------|
| ENOENT (EN) ↔ ENOENT (KR) | 0.6578 |
| ENOENT (EN) ↔ EACCES (EN) | 0.6160 |
| ENOENT (EN) ↔ "관련 없는" | 0.9387 |

**결론**: 영어 간 구분은 좋지만, 한국어를 전혀 이해하지 못함. 부적합.

---

### 2.6 Ollama — nomic-embed-text (미테스트)

| 항목 | 값 |
|------|-----|
| 모델 | nomic-embed-text v2 |
| 차원 | 768 |
| 모델 크기 | ~274 MB |
| 다국어 | ~100 언어 |
| 요구사항 | Ollama 데몬 실행 필요 (brew install ollama) |
| 설치 상태 | ❌ 미설치 |

**비고**: brew로 설치 가능 (ollama 0.15.5). GPU 가속이 가능하여 고품질 임베딩 기대. 단, 항상 실행 데몬이 필요하여 훅 환경에서 cold start 문제 있을 수 있음.

---

## 3. 모델 비교 종합

| 모델 | 차원 | 에러KB F1 | 최적 임계값 | 분포 폭 | Top-1 정확도 | 한국어 | 메모리 | 속도 |
|------|-----|-----------|-----------|---------|-------------|--------|--------|------|
| **paraphrase-multilingual** | 384 | **1.00** | **0.76** | **0.12~0.93** | **6/6** | **양호** | 717MB | 2.4ms |
| multilingual-e5-small | 384 | 0.80 | 0.17 | 0.05~0.22 | 4/6 | 보통 | 689MB | 4~8ms |
| bge-small-en-v1.5 | 384 | 0.80 | 0.50 | 0.14~0.64 | 5/6 | 미흡 | 1158MB | 3.7ms |
| gte-small | 384 | 0.80 | 0.20 | 0.06~0.26 | 5/6 | 미흡 | 1159MB | 2.3ms |
| all-MiniLM-L6-v2 | 384 | N/A | N/A | 0.34~0.94 | N/A | ❌ | - | - |

### 승자: `paraphrase-multilingual-MiniLM-L12-v2`

**결정적 차이점**:
1. **에러 KB F1 = 1.00** (다른 모델은 모두 0.80)
2. **Top-1 정확도 6/6** — 한국어 순수 텍스트("파일을 찾을 수 없습니다", "데이터베이스 연결이 안됩니다") 포함 모든 쿼리 정확
3. **분포 폭 0.81** (0.12~0.93) — 임계값 설정 매우 용이, 안전 마진 0.14
4. **프리픽스 불필요** — 구현 단순화
5. **가장 빠름** — 2.4ms/건

---

## 4. 비용 비교

| 솔루션 | 월 비용 | API Key 필요 | 외부 의존성 |
|--------|---------|:------------:|:-----------:|
| Transformers.js | 무료 | ❌ | ❌ |
| Ollama | 무료 | ❌ | Ollama 데몬 |
| Voyage AI | 무료 (200M 토큰) | ✅ | 인터넷 |
| OpenAI | ~$0.01 | ✅ | 인터넷 |

---

## 5. 최종 추천 전략

### Primary: Transformers.js + paraphrase-multilingual-MiniLM-L12-v2

```javascript
// 패키지: @xenova/transformers (유일한 외부 의존성)
// 모델: Xenova/paraphrase-multilingual-MiniLM-L12-v2
// 차원: 384
// 임계값: 0.76 (고신뢰), 0.80 (저신뢰 + 키워드 검증)

import { pipeline } from '@xenova/transformers';
const extractor = await pipeline('feature-extraction',
  'Xenova/paraphrase-multilingual-MiniLM-L12-v2');

async function embed(text) {
  const out = await extractor(text, { pooling: 'mean', normalize: true });
  return Array.from(out.data); // Float32Array → Array for sqlite-vec
}
```

### Fallback: FTS5 키워드 매칭

벡터 검색 실패 시 (모델 로드 오류 등) SQLite FTS5로 폴백:

```sql
SELECT * FROM error_kb_fts WHERE error_kb_fts MATCH ?;
```

### 임계값 전략

```
distance < 0.76  → 고신뢰 매칭 (바로 반환)
0.76 ~ 0.85      → 저신뢰 (키워드 검증 후 반환)
distance >= 0.85  → 매칭 없음
```

### 스킬 매칭 대응

paraphrase 모델의 한국어→영어 의미 검색 약점을 보완:
- 스킬 설명을 **이중 언어**로 저장: "Docker 이미지 빌드 / Build Docker images"
- 또는 스킬별 한국어 키워드 목록 추가

### 메모리 최적화

| 방식 | 메모리 | 지연 |
|------|--------|------|
| 훅마다 로드 | 717MB × N | 3.7s cold start |
| **장기 프로세스** | **717MB × 1** | **<3ms** |
| 사전 계산 + 캐시 | 0 (런타임 불필요) | 0 |

**추천**: 임베딩을 **데이터 저장 시점에 사전 계산**하여 SQLite에 벡터로 저장.
검색 시에는 쿼리 임베딩만 생성 (단발성이므로 cold start 감수 가능, 또는 사전 로드된 프로세스 활용).

### DESIGN.md 반영 사항

| 항목 | 기존 (DESIGN.md) | 변경 |
|------|------------------|------|
| 임베딩 생성 | `claude --print` | Transformers.js `paraphrase-multilingual-MiniLM-L12-v2` |
| 차원 | 384 | 384 (동일) |
| 임계값 | 0.3 | 0.76 |
| 외부 의존성 | 없음 | `@xenova/transformers` 1개 추가 |
| 프리픽스 | query:/passage: | 불필요 |

---

## 6. 테스트 코드 위치

- `/tmp/embedding-test/test.mjs` — 기본 성능 테스트 (e5-small)
- `/tmp/embedding-test/test2.mjs` — 유사도 분포 테스트 (e5-small)
- `/tmp/embedding-test/test3.mjs` — 모델 비교, 랭킹 전략 (e5 vs MiniLM)
- `/tmp/embedding-test/test4.mjs` — 실전 Error KB 시뮬레이션 + 임계값 탐색 (e5-small)
- `/tmp/embedding-test/test5-multilingual-models.mjs` — 3모델 비교 (paraphrase, bge, gte)
- `/tmp/embedding-test/test6-paraphrase-detail.mjs` — paraphrase 모델 세밀 분석 + 하이브리드 전략
