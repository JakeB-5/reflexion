# QMD 분석: 아키텍처 & 실측 벤치마크

> **qmd** — 로컬 온디바이스 검색 엔진. BM25 + Vector + LLM Reranking을 조합하여 마크다운/코드 문서를 검색한다.
> 모든 LLM 연산이 로컬 GGUF 모델로 실행되며 외부 API 호출이 없다.
>
> GitHub: [tobi/qmd](https://github.com/tobi/qmd)

---

## 1. 아키텍처 개요

### 모듈 의존성

```
qmd.ts (CLI entry point, 97KB)
  ├── store.ts (88KB) ── Core storage, search, indexing, RRF fusion
  │     ├── llm.ts (36KB) ── node-llama-cpp wrapper (embed, generate, rerank)
  │     └── collections.ts (9KB) ── YAML config management
  ├── formatter.ts (13KB) ── Multi-format output (JSON/CSV/XML/MD/CLI)
  └── mcp.ts (24KB) ── MCP server (stdio transport)
        └── store.ts (reuses same store)
```

### 계층 구조

| Layer | Module | 역할 |
|-------|--------|------|
| Configuration | `collections.ts` | YAML 기반 컬렉션 정의 (`~/.config/qmd/index.yml`) |
| Data | `store.ts` | SQLite + sqlite-vec, CRUD, 검색, 청킹, RRF |
| LLM | `llm.ts` | node-llama-cpp 추상화, 3개 GGUF 모델 관리 |
| Interface | `qmd.ts`, `mcp.ts` | CLI와 MCP 서버 — store/LLM API 소비자 |
| Presentation | `formatter.ts` | 6개 출력 포맷 (JSON, CSV, XML, MD, files, CLI) |

### 핵심 설계 패턴

- **Store Factory** (`createStore()`): 글로벌 싱글톤 없이 DB 연결 캡슐화
- **Content-Addressable Storage**: SHA-256 해시 기반 문서 중복 제거
- **Virtual Path System**: `qmd://collection/path`로 파일시스템 추상화
- **Scoped Session** (`withLLMSession()`): 참조 카운팅 + abort signal + max duration timer로 리소스 누수 방지
- **Lazy Loading + Promise Guard**: 모델 로딩 중복 방지 (VRAM 이중 할당 차단)

---

## 2. 로컬 GGUF 모델

| Model | 용도 | 크기 |
|-------|------|------|
| `embeddinggemma-300M-Q8_0` | 벡터 임베딩 | ~300MB |
| `qmd-query-expansion-1.7B-q4_k_m` | 쿼리 확장 (fine-tuned Qwen3) | ~1.1GB |
| `qwen3-reranker-0.6b-q8_0` | Cross-encoder 리랭킹 | ~640MB |

모델은 HuggingFace에서 자동 다운로드되며 `~/.cache/qmd/models/`에 캐싱된다.

---

## 3. 데이터 파이프라인

### 인덱싱 흐름

```
Collection 등록 → Glob 패턴 매칭 → 파일 읽기 → SHA-256 해시
  → Title 추출 (첫 # heading or filename)
  → content 테이블에 INSERT OR IGNORE (해시 기반 중복 방지)
  → documents 테이블에 INSERT/UPDATE (collection + path → hash 매핑)
  → 디스크에서 삭제된 문서 비활성화 (active=0)
```

### 임베딩 흐름

```
content 테이블에서 임베딩 미생성 해시 조회
  → 800-token 청킹 (15% = 120 token 오버랩)
  → 스마트 분할 (paragraph > sentence > line > word boundary)
  → 청크 포맷: "title: {title} | text: {chunk_text}"
  → 32개씩 배치 임베딩 (session.embedBatch())
  → content_vectors (메타데이터) + vectors_vec (벡터) 저장
```

---

## 4. 검색 파이프라인

### 4.1 BM25 Search (`search`)

```
Query → sanitizeFTS5Term() → buildFTS5Query()
  → FTS5 MATCH with bm25(10.0, 1.0) weighting
  → Score normalization: 1/(1+|bm25|)
  → SearchResult[]
```

- filepath에 10x 가중치, body에 1x
- LLM 불필요 — 순수 SQLite FTS5

### 4.2 Vector Search (`vsearch`)

```
Query → formatQueryForEmbedding("task: search result | query: {query}")
  → LLM embed() → sqlite-vec KNN (k=limit×3)
  → Two-step JOIN (sqlite-vec JOIN 버그 회피)
  → filepath 기준 중복 제거
  → Score = 1 - cosine_distance
```

- Two-phase query가 필수: sqlite-vec에서 JOIN 하면 무한 대기 발생

### 4.3 Hybrid Query (`query`) — 전체 파이프라인

```
Step 1: 초기 BM25 → score ≥ 0.85이면 쿼리 확장 스킵 (strong-signal shortcut)
Step 2: LLM 쿼리 확장 → lexical/vector/HyDE 변형 (GBNF 문법 제약)
Step 3: 모든 변형에 대해 FTS + Vector 병렬 검색
Step 4: RRF 융합 (k=60)
         - Original query 결과에 2x 가중치
         - Top-rank bonus: #1 +0.05, #2-3 +0.02
Step 5: 문서별 최적 청크 선택 (키워드 매칭 기준)
Step 6: LLM 리랭킹 (qwen3-reranker, 최대 40문서)
Step 7: Position-aware blending:
         - Rank 1-3:  75% RRF / 25% reranker
         - Rank 4-10: 60% RRF / 40% reranker
         - Rank 11+:  40% RRF / 60% reranker
```

---

## 5. SQLite 스키마

- **4 regular tables**: collections, documents, content (SHA-256 key), content_vectors, path_contexts, llm_cache
- **2 virtual tables**: documents_fts (FTS5), vectors_vec (sqlite-vec)
- **3 triggers**: FTS 동기화 (INSERT/UPDATE/DELETE)
- **WAL mode**: 동시 읽기/쓰기 지원

---

## 6. MCP 통합

| 도구 | 기능 |
|------|------|
| `search` | BM25 키워드 검색 |
| `vsearch` | 벡터 시맨틱 검색 |
| `query` | 하이브리드 검색 (최고 품질) |
| `get` | 문서 전체 내용 조회 |
| `multi_get` | 여러 문서 한번에 조회 |
| `status` | 인덱스 상태 확인 |

- `@modelcontextprotocol/sdk` + `StdioServerTransport`
- 1 resource template (`qmd://{+path}`), 1 prompt (query guide)
- DB는 시작 시 1회 열고 재사용

---

## 7. 실측 벤치마크

### 테스트 환경

| 항목 | 값 |
|------|-----|
| 프로젝트 | reflexion |
| 문서 | 4개 (DESIGN.md, CLAUDE.md, RESULT.md, REFERENCE-HOOK-SYSTEM.md) |
| 임베딩 | 67개 벡터 |
| DB 크기 | 3.5 MB |
| 패턴 | `**/*.md` |
| 플랫폼 | macOS (Apple Silicon) |

### 7.1 BM25 검색 (5개 영문 키워드 쿼리)

| Query | Time | Results | Top Score |
|-------|------|---------|-----------|
| "SQLite schema" | 0.174s | 1 | 0.44 |
| "hook event" | 0.173s | 2 | 1.00 |
| "embedding server" | 0.195s | 2 | 1.00 |
| "feedback tracker" | 0.184s | 3 | 1.00 |
| "session analysis" | 0.173s | 2 | 1.00 |
| **평균** | **0.180s** | - | **0.848** |

### 7.2 벡터 검색 (5개 한국어 시맨틱 쿼리)

| Query | Time | Results | Top Score |
|-------|------|---------|-----------|
| "에러가 발생했을 때 처리 방법" | 15.234s | 3 | 0.62 |
| "데이터베이스 구조 설계" | 4.390s | 3 | 0.54 |
| "AI 분석 파이프라인" | 4.289s | 3 | 0.59 |
| "사용자 프롬프트 수집" | 4.311s | 3 | 0.60 |
| "성능 최적화 전략" | 3.796s | 3 | 0.59 |
| **평균 (steady)** | **4.197s** | - | **0.588** |

> 첫 쿼리 15.2s는 모델 로딩 포함 (cold start). Steady-state 평균 4.2s.

### 7.3 하이브리드 검색 (3개 한국어 자연어 쿼리)

| Query | Time | Results | Top Score | Pipeline |
|-------|------|---------|-----------|----------|
| "세션이 끝나면 어떤 분석이 실행되나" | 20.529s | 3 | 0.92 | 3 lex + 4 vec + HyDE → rerank 4 |
| "에러 해결 이력을 어떻게 추적하나" | 8.563s | 3 | 0.88 | 3 lex + 4 vec + HyDE → rerank 4 |
| "스킬 자동 생성 워크플로우" | 8.167s | 3 | 0.93 | 3 lex + 4 vec + HyDE → rerank 4 |
| **평균 (steady)** | **8.365s** | - | **0.910** | |

> 첫 쿼리 20.5s는 모든 모델 로딩 (embedding + expansion + reranker). Steady-state 평균 8.4s.

---

## 8. 성능 비교 요약

| Method | Avg Time (steady) | vs BM25 | Avg Top Score | 용도 |
|--------|-------------------|---------|---------------|------|
| **BM25** | 0.180s | 1x | 0.848 | 정확한 키워드, 속도 중요 |
| **Vector** | 4.197s | 23x slower | 0.588 | 시맨틱 검색, 개념 탐색 |
| **Hybrid** | 8.365s | 46x slower | 0.910 | 최고 품질, 탐색적 리서치 |

### Cold Start 페널티

| Mode | 첫 쿼리 | Steady-state | 배율 |
|------|---------|-------------|------|
| Vector | 15.2s | 4.2s | 4x |
| Hybrid | 20.5s | 8.4s | 2.5x |

---

## 9. 내부 최적화 전략

| 전략 | 설명 |
|------|------|
| Lazy model loading + Promise guard | 모델 중복 로딩 방지, VRAM 이중 할당 차단 |
| 5분 idle timeout | 미사용 모델 자동 해제 |
| LLM result cache | `llm_cache` 테이블에 쿼리 확장/리랭킹 결과 캐싱 |
| Content-addressable storage | 변경 없는 콘텐츠 재임베딩 스킵 |
| Strong-signal shortcut | BM25 score >= 0.85이면 쿼리 확장 생략 |
| Batch embedding | 32개 청크씩 일괄 처리 |
| Two-phase vector query | sqlite-vec JOIN 버그 회피용 2단계 쿼리 |
| Reranking cap | 최대 40문서만 리랭킹 (비용 제한) |

---

## 10. reflexion 프로젝트와의 비교

| 항목 | qmd | reflexion (DESIGN.md) |
|------|-----|---------------------------|
| Embedding 모델 | embeddinggemma-300M (GGUF) | paraphrase-multilingual-MiniLM-L12-v2 |
| Embedding 런타임 | node-llama-cpp | @xenova/transformers |
| 벡터 차원 | dynamic | 384 |
| 벡터 저장 | sqlite-vec | sqlite-vec |
| 청킹 전략 | 800 tokens, 15% overlap | 미정 |
| FTS | FTS5 | FTS5 |
| 리랭킹 | qwen3-reranker (local) | 없음 |
| Query expansion | Fine-tuned Qwen3 1.7B | 없음 |
| DB 모드 | WAL | WAL |

**적용 가능한 패턴:**
- sqlite-vec two-phase query 패턴 (JOIN 버그 회피)
- Content-addressable storage (해시 기반 중복 방지)
- FTS5 trigger 기반 동기화
- 800-token / 15% overlap 청킹 전략
- Strong-signal shortcut (높은 BM25 점수 시 벡터 검색 생략)

---

## 11. 제약사항

- **소규모 데이터셋**: 4개 문서, 67개 임베딩으로 테스트 — 1000+ 문서 환경에서 성능 특성이 다를 수 있음
- **캐시 효과**: 반복 쿼리 시 파일시스템 캐시로 시간이 과소 측정될 수 있음
- **한국어 중심**: 벡터/하이브리드 테스트는 한국어 쿼리만 사용
- **단일 세션**: 다중 동시 사용자 시나리오 미검증
- **Cold start 1회**: 각 모드당 첫 쿼리만 측정

---

*측정일: 2026-02-08*
