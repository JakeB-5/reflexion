# Reflexion vs Claude-Mem vs QMD: 3개 시스템 비교 연구

> 작성일: 2026-02-08
> 목적: reflexion 설계에 참고할 수 있는 패턴, 도입 가능성, 구조 단순화 방안 도출

---

## 1. 프로젝트 정체성 비교

| 항목 | reflexion | claude-mem | QMD |
|------|----------------|-----------|-----|
| **한줄 요약** | 프롬프트 패턴 분석 → 자동 개선 제안 | 세션 간 메모리 보존 | 로컬 온디바이스 하이브리드 검색 엔진 |
| **핵심 질문** | "사용자가 뭘 반복하나?" | "이전 세션에서 뭘 했나?" | "이 문서에서 뭘 찾나?" |
| **대상** | Claude Code 사용자 행동 | Claude Code 세션 컨텍스트 | 마크다운/코드 문서 |
| **출력** | 스킬, CLAUDE.md 지침, 훅 워크플로우 | 이전 세션 컨텍스트 주입 | 검색 결과 (ranked documents) |
| **작동 시점** | 세션 간(배치) + 세션 내(실시간) | 세션 시작/종료/도중 | 온디맨드 (CLI/MCP 호출 시) |
| **AI 의존성** | `claude --print` (배치 분석) | Claude Agent SDK (요약/압축) | 로컬 GGUF 3개 (임베딩, 확장, 리랭킹) |
| **배포 방식** | 독립 설치 (`~/.reflexion/`) | Claude Code Plugin Marketplace | 독립 CLI + MCP Server |

### 관계 다이어그램

```
                    ┌─────────────────┐
                    │   Claude Code   │
                    │    Session      │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌────────────┐  ┌─────────────┐  ┌─────────┐
     │ reflexion  │  │ claude-mem  │  │   QMD   │
     │            │  │             │  │         │
     │ "왜 반복   │  │ "뭘 했었지" │  │ "어디에 │
     │  하는가?"  │  │  기억하기   │  │  있지?" │
     └─────┬──────┘  └──────┬──────┘  └────┬────┘
           │                │               │
           ▼                ▼               ▼
     개선 제안 생성    컨텍스트 보존     문서 검색 결과
     (스킬/규칙/훅)   (세션 간 연속성)  (BM25/Vector/Hybrid)
```

---

## 2. 데이터 수집 비교

### 2.1 수집 대상

| 수집 항목 | reflexion | claude-mem | QMD |
|----------|----------------|-----------|-----|
| 사용자 프롬프트 | O (전문) | X (미수집) | X (해당없음) |
| 도구 사용 | O (도구명+메타) | O (**관찰** 단위) | X |
| 도구 에러 | O (에러+해결 추적) | X | X |
| 도구 응답 | X (프라이버시) | O (압축 후 저장) | X |
| 세션 요약 | O (AI 생성) | O (AI 생성) | X |
| 서브에이전트 | O (성능 메트릭) | X | X |
| 외부 문서 | X | X | O (마크다운/코드 파일) |

### 2.2 수집 메커니즘

| 메커니즘 | reflexion | claude-mem | QMD |
|---------|----------------|-----------|-----|
| **트리거** | Hook 스크립트 (직접 실행) | Hook → Worker HTTP 위임 | CLI 명령어 (수동 인덱싱) |
| **처리 방식** | 훅 내 동기 DB write | 비동기 Worker 큐 | 배치 인덱싱 |
| **비차단 보장** | exit 0 + try/catch | Worker 위임 + exit 0 | 해당없음 (CLI) |
| **타임아웃** | 2-5s (훅별) | 30-120s (Worker) | 없음 |

### 2.3 핵심 차이

**reflexion**은 "**무엇을 왜 했는가**"(프롬프트 의도 + 도구 패턴)를 수집하고,
**claude-mem**은 "**무엇이 일어났는가**"(도구 사용 관찰의 결과)를 수집하며,
**QMD**는 수집이 아닌 "**이미 있는 문서를 인덱싱**"한다.

---

## 3. DB 활용 비교

### 3.1 저장소 아키텍처

| 항목 | reflexion | claude-mem | QMD |
|------|----------------|-----------|-----|
| **메인 DB** | SQLite (`better-sqlite3`) | SQLite (Bun 내장) | SQLite (`better-sqlite3`) |
| **벡터 저장** | `sqlite-vec` (float[384]) | Chroma Vector DB (별도) | `sqlite-vec` (동적 차원) |
| **FTS** | 미사용 (벡터 검색 의존) | FTS5 | FTS5 + BM25 커스텀 가중치 |
| **DB 모드** | WAL | 미명시 (SQLite 기본) | WAL |
| **DB 파일** | 단일 (`reflexion.db`) | 단일 (`claude-mem.db`) + Chroma 별도 | 단일 (컬렉션별) |
| **동시성** | `busy_timeout = 5000` | Worker 직렬화 | WAL |

### 3.2 테이블 구조 비교

**reflexion (5 regular + 2 vec0 virtual)**
```
events              ← 전역 이벤트 로그 (prompt, tool_use, tool_error, ...)
error_kb            ← 에러 해결 이력 + 정규화
feedback            ← 제안 채택/거부
analysis_cache      ← AI 분석 결과
skill_embeddings    ← 스킬 벡터
vec_error_kb        ← 에러 벡터 (sqlite-vec)
vec_skill_embeddings ← 스킬 벡터 (sqlite-vec)
```

**claude-mem (추정 — SQLite + Chroma)**
```
sessions            ← 세션 메타데이터
observations        ← 도구 사용 관찰 (압축된 내용)
summaries           ← AI 생성 세션 요약
---
Chroma Collection   ← 벡터 임베딩 (별도 프로세스)
```

**QMD (4 regular + 2 virtual + 3 triggers)**
```
collections         ← 컬렉션 설정
documents           ← 파일 경로 → 해시 매핑
content             ← SHA-256 기반 콘텐츠 (중복 제거)
content_vectors     ← 청크 메타데이터
llm_cache           ← 쿼리 확장/리랭킹 캐시
documents_fts       ← FTS5 가상 테이블
vectors_vec         ← sqlite-vec 가상 테이블
+ INSERT/UPDATE/DELETE triggers (FTS 동기화)
```

### 3.3 벡터 검색 비교

| 항목 | reflexion | claude-mem | QMD |
|------|----------------|-----------|-----|
| **임베딩 모델** | MiniLM-L12-v2 (ONNX) | Chroma 내장 | embeddinggemma-300M (GGUF) |
| **차원** | 384 | Chroma 의존 | 동적 |
| **런타임** | Transformers.js (Node) | Chroma (Python/uv) | node-llama-cpp |
| **인프라** | Unix socket 데몬 | Chroma 서버 (별도 프로세스) | 인프로세스 |
| **모델 크기** | ~120MB ONNX | Chroma 의존 | ~300MB GGUF |
| **속도** | ~2.4ms/건 (데몬) | 미공개 | ~4.2s/쿼리 (steady) |
| **검색 방식** | KNN (2단계 쿼리) | Chroma native | KNN (2단계 쿼리) |
| **리랭킹** | 없음 | 없음 | qwen3-reranker (로컬) |
| **쿼리 확장** | 없음 | 없음 | Fine-tuned Qwen3 1.7B |
| **하이브리드** | 없음 | FTS5 + Chroma | BM25 + Vector + RRF 융합 |

---

## 4. 공통점

### 4.1 아키텍처적 공통점

| 공통 패턴 | 세부 |
|----------|------|
| **Claude Code Hooks 활용** | reflexion과 claude-mem 모두 Hooks API로 데이터 수집 |
| **SQLite 기반 저장** | 3개 프로젝트 모두 SQLite를 메인 DB로 사용 |
| **sqlite-vec 활용** | reflexion과 QMD 모두 sqlite-vec으로 벡터 검색 |
| **WAL 모드** | 3개 프로젝트 모두 동시성을 위해 WAL 사용 |
| **비차단 설계** | reflexion과 claude-mem 모두 훅 실행 시 세션 차단 방지 |
| **MCP 통합** | claude-mem과 QMD 모두 MCP Server로 검색 인터페이스 제공 |
| **전역 설치** | 3개 프로젝트 모두 `~/` 하위에 전역 데이터 저장 |
| **2단계 벡터 쿼리** | reflexion과 QMD 모두 sqlite-vec JOIN 버그 회피용 2단계 쿼리 사용 |

### 4.2 설계 철학 공통점

- **로컬 우선**: 외부 클라우드 서비스 최소화 (특히 reflexion, QMD)
- **프라이버시 우선**: 민감 데이터 보호 장치 (reflexion: Bash 첫 단어만, claude-mem: `<private>` 태그)
- **점진적 발전**: 단순한 수집 → 분석 → 활용으로 단계적 구현

---

## 5. 각 시스템의 데이터 저장/활용 특징

### 5.1 reflexion: 패턴 중심 저장

```
수집 (hooks) → events 테이블 (원시 데이터)
                    │
                    ▼
            AI 분석 (claude --print, 세션 종료 시)
                    │
                    ▼
            analysis_cache (분석 결과 캐시)
                    │
                    ▼
            다음 SessionStart에서 주입
                    │
                    ▼
         사용자 승인 → 스킬/규칙/훅 생성
```

**특징:**
- 원시 이벤트 → AI 분석 → 제안의 **3단계 파이프라인**
- 에러 KB는 벡터 유사도로 실시간 검색 (세션 내)
- 스킬 매칭도 벡터 유사도로 실시간 (프롬프트 수신 시)
- 분석은 `claude --print`에 의존 (API 비용 발생)

### 5.2 claude-mem: 관찰 중심 저장

```
도구 사용 (PostToolUse) → Worker 큐 → observation 저장
                                            │
                                            ▼
                              Stop 훅 → AI 요약 (Claude Agent SDK)
                                            │
                                            ▼
                              summary 저장 + Chroma 벡터 동기화
                                            │
                                            ▼
                              다음 SessionStart에서 컨텍스트 주입
                                            │
                                            ▼
                              MCP 도구로 온디맨드 검색
```

**특징:**
- **도구 응답 내용**까지 캡처하여 "무엇이 일어났는지" 기록
- Worker Service가 모든 무거운 처리를 비동기로 담당
- Progressive Disclosure: 검색 시 인덱스 → 타임라인 → 상세 3단계
- Chroma + FTS5 이중 검색 인프라

### 5.3 QMD: 문서 중심 저장

```
파일 시스템 → Glob 패턴 매칭 → SHA-256 해시
                                    │
                                    ▼
                        content 테이블 (중복 제거)
                                    │
                        ┌───────────┼───────────┐
                        ▼           ▼           ▼
                    FTS5 인덱스  청킹+임베딩   documents 매핑
                        │           │
                        ▼           ▼
                     BM25 검색   Vector 검색
                        │           │
                        └─────┬─────┘
                              ▼
                        RRF 융합 + 리랭킹
```

**특징:**
- **Content-Addressable Storage**: SHA-256으로 변경 없는 콘텐츠 재처리 방지
- 3개 로컬 GGUF 모델로 완전 오프라인 검색
- 가장 정교한 검색 파이프라인 (BM25 + Vector + RRF + Reranking)
- Strong-signal shortcut: BM25 점수 높으면 벡터 검색 스킵

---

## 6. reflexion에 차용할 수 있는 패턴

### 6.1 claude-mem에서 차용

| 패턴 | 설명 | 적용 방안 | 난이도 |
|------|------|----------|--------|
| **Worker 위임 패턴** | 훅이 직접 처리하지 않고 상주 Worker에 HTTP 위임 | 현재 임베딩 데몬과 유사하나, 모든 무거운 처리를 Worker로 통합 가능 | 중 |
| **Progressive Disclosure** | 검색 결과를 인덱스→상세로 점진 조회 | 에러 KB/스킬 매칭 결과를 2단계로 나눠 토큰 절약 | 하 |
| **MCP 검색 인터페이스** | 메모리를 MCP 도구로 노출 | 분석 결과/에러 KB를 MCP로 노출하면 Claude가 자연어로 검색 가능 | 중 |
| **Smart Install** | 의존성 캐시 체크로 불필요한 재설치 방지 | 첫 실행 시 `better-sqlite3`, ONNX 모델 설치 체크에 적용 | 하 |
| **Privacy 태그** | `<private>` 태그로 민감 정보 제외 | 프롬프트 수집 시 `<private>` 태그 내용 스트리핑 | 하 |

### 6.2 QMD에서 차용

| 패턴 | 설명 | 적용 방안 | 난이도 |
|------|------|----------|--------|
| **Content-Addressable Storage** | SHA-256 해시 기반 중복 방지 | 분석 결과 캐시에 입력 해시를 키로 사용하여 동일 입력 재분석 방지 | 하 |
| **FTS5 + 벡터 하이브리드** | BM25와 벡터를 RRF로 융합 | 에러 KB 검색에 FTS5 추가하여 키워드 정확 매칭 보완 | 중 |
| **Strong-signal shortcut** | 높은 BM25 점수 시 벡터 검색 스킵 | 에러 KB에서 정확 매칭 발견 시 벡터 검색 생략 → 속도 개선 | 하 |
| **FTS5 trigger 동기화** | INSERT/UPDATE/DELETE trigger로 FTS 자동 동기화 | events 테이블에 FTS5 추가 시 trigger 방식으로 동기화 | 하 |
| **LLM 결과 캐시** | `llm_cache` 테이블에 쿼리 확장/리랭킹 결과 저장 | `analysis_cache`에 이미 유사 패턴 있으나, 에러 KB 검색 결과도 캐싱 가능 | 하 |
| **Batch embedding** | 32개씩 일괄 임베딩 | 현재 SessionEnd 배치 임베딩에 적용 가능 (chunk size 최적화) | 하 |
| **800-token 청킹 + 15% overlap** | 스마트 분할 전략 | 프롬프트/에러 텍스트가 긴 경우 청킹 전략으로 임베딩 품질 향상 | 중 |

---

## 7. 설계 대체 검토

### 7.1 현재 reflexion의 임베딩 인프라 vs 대안

| 현재 설계 | 대안 A: QMD 방식 | 대안 B: claude-mem 방식 |
|----------|----------------|---------------------|
| Transformers.js 데몬 (Unix socket) | node-llama-cpp 인프로세스 | Chroma 별도 프로세스 |
| MiniLM-L12-v2 (ONNX, 120MB) | embeddinggemma-300M (GGUF, 300MB) | Chroma 내장 모델 |
| 384차원, ~2.4ms/건 | 동적 차원, ~4.2s/쿼리 | 미공개 |
| sqlite-vec 직접 | sqlite-vec 직접 | Chroma API |
| **장점**: 빠름, 가벼움, 다국어 | **장점**: 더 나은 임베딩 품질, 리랭킹 가능 | **장점**: 관리 편의 |
| **단점**: 리랭킹/쿼리 확장 없음 | **단점**: 느림, VRAM 필요 | **단점**: Python 의존, 무거움 |

**결론**: 현재 설계가 reflexion의 용도(에러 KB/스킬 매칭)에 최적. QMD 수준의 정교한 검색은 불필요.

### 7.2 현재 데이터 수집 vs claude-mem Worker 패턴

| 현재 설계 | Worker 패턴 대체 |
|----------|-----------------|
| 각 훅이 직접 SQLite write | 훅 → HTTP POST → Worker가 처리 |
| 훅 내 DB 연결/해제 반복 | Worker가 DB 연결 유지 (커넥션 풀) |
| 훅 실행 시간: ~10-50ms | 훅 실행 시간: ~5ms (HTTP POST만) |
| 단순, 의존성 없음 | Express/Bun 추가 의존 |

**결론**: reflexion의 훅은 이미 충분히 가볍다 (동기 SQLite write ~10ms). Worker 패턴은 claude-mem처럼 AI 요약 등 무거운 처리가 훅 내에서 필요할 때 가치가 있다. reflexion은 무거운 처리를 SessionEnd의 detached 프로세스로 분리하므로 현재 설계가 적합.

### 7.3 MCP 검색 인터페이스 추가 검토

| 현재 설계 | MCP 추가 시 |
|----------|------------|
| SessionStart에서 분석 캐시 일괄 주입 | Claude가 필요할 때 MCP로 검색 |
| 고정된 컨텍스트 주입 | 동적, 질문 맥락에 맞는 검색 |
| 토큰 낭비 가능 (불필요한 컨텍스트) | 토큰 효율적 (필요한 것만) |
| 추가 인프라 불필요 | MCP Server 실행 필요 |

**결론**: Phase 5 이후 확장으로 MCP 검색 인터페이스 추가는 매우 유용. 특히 에러 KB 검색을 MCP로 노출하면, PostToolUseFailure 훅의 자동 검색 + Claude의 능동적 검색이 모두 가능해진다.

---

## 8. 종속성 도입으로 구조 단순화 가능성

### 8.1 claude-mem을 종속성으로 도입

```
시나리오: claude-mem이 "도구 사용 관찰 + 세션 요약"을 이미 처리하므로,
         reflexion은 분석/제안에만 집중
```

| 영역 | claude-mem에 위임 | reflexion이 유지 |
|------|------------------|---------------------|
| 도구 사용 수집 | O (PostToolUse observation) | X (제거) |
| 세션 요약 | O (Stop → summarize) | X (제거) |
| 프롬프트 수집 | X (claude-mem 미수집) | O (유지 필요) |
| 에러 추적 | X (claude-mem 미수집) | O (유지 필요) |
| 서브에이전트 추적 | X (claude-mem 미수집) | O (유지 필요) |
| 패턴 분석 | X (claude-mem은 분석 안 함) | O (핵심 기능) |
| 제안 생성 | X | O (핵심 기능) |
| 피드백 추적 | X | O (핵심 기능) |
| 에러 KB | X | O (핵심 기능) |
| 스킬 매칭 | X | O (핵심 기능) |

**평가:**
- **제거 가능**: `tool-logger.mjs`, `session-summary.mjs`의 일부 기능 (도구 사용 기록, 세션 요약)
- **제거 불가**: 프롬프트 수집, 에러 추적, 해결 패턴 감지, 패턴 분석, 제안 엔진 전체
- **문제점**:
  - claude-mem의 데이터 스키마가 reflexion의 분석에 맞지 않음 (observation vs event)
  - claude-mem은 "무엇이 일어났나"를 기록하지만, reflexion은 "왜/어떤 패턴으로"를 분석해야 함
  - claude-mem의 MCP 검색으로 데이터를 가져와도, 원시 이벤트 형태가 아니라 요약/압축된 형태
  - AGPL-3.0 라이선스 오염 위험

**결론: 종속성 도입 부적합**. 수집 대상과 데이터 형태가 근본적으로 다르다. reflexion이 필요한 원시 이벤트 데이터는 claude-mem에서 제공하지 않는다.

### 8.2 QMD를 종속성으로 도입

```
시나리오: QMD의 검색 엔진을 에러 KB 검색이나 문서 검색에 활용
```

| 영역 | QMD에 위임 가능 | 비고 |
|------|---------------|------|
| 에러 KB 벡터 검색 | △ (가능하나 과잉) | QMD는 문서 검색용, 에러 메시지 검색에는 과잉 설계 |
| 스킬 매칭 | △ (가능하나 과잉) | 동일 이유 |
| DESIGN.md 등 참조 문서 검색 | O (적합) | QMD의 본래 용도 |
| 분석 결과 검색 | △ | 구조화된 JSON이라 문서 검색보다 SQL이 적합 |

**평가:**
- QMD는 **문서 검색 엔진**이지, 구조화된 이벤트 데이터 검색에는 부적합
- reflexion의 벡터 검색은 단순 KNN (에러 유사도, 스킬 매칭)이므로 QMD의 복잡한 파이프라인이 필요 없음
- QMD를 함께 실행하면 GGUF 모델 3개 (~2GB) 추가 메모리 부담
- 다만, reflexion이 생성한 분석 리포트나 스킬 문서를 QMD로 인덱싱하면 "내가 만든 자동화의 이력"을 검색하는 데 유용할 수 있음

**결론: 종속성 도입 부적합, 보완적 활용은 가능**. QMD는 독립적으로 운영하면서 reflexion이 생성한 문서를 QMD 컬렉션에 추가하는 방식이 자연스럽다.

### 8.3 추천: 독립 운영 + 인터페이스 연동

```
┌─────────────────┐     ┌─────────────────┐     ┌──────────────┐
│    reflexion    │     │   claude-mem    │     │     QMD      │
│                 │     │                 │     │              │
│ 패턴 분석/제안   │     │ 세션 메모리 보존  │     │ 문서 검색     │
│                 │     │                 │     │              │
│ ~/.reflexion/   │     │ ~/.claude-mem/  │     │ ~/.config/qmd│
│ reflexion.db    │     │ claude-mem.db   │     │ DB files     │
└────────┬────────┘     └────────┬────────┘     └──────┬───────┘
         │                       │                      │
         │    ┌──────────────────┼──────────────────────┘
         │    │                  │
         ▼    ▼                  ▼
    ┌─────────────────────────────────────┐
    │         Claude Code Session          │
    │                                     │
    │  Hooks: reflexion + claude-mem      │
    │  MCP: claude-mem search + QMD      │
    │  Context: reflexion 분석 + mem 기억 │
    └─────────────────────────────────────┘
```

**3개 시스템은 독립적으로 운영하되 Claude Code 세션에서 자연스럽게 합류한다:**
- **reflexion**: Hooks로 패턴 수집/분석, SessionStart에서 개선 제안 주입
- **claude-mem**: Hooks로 메모리 보존, SessionStart에서 이전 컨텍스트 주입
- **QMD**: MCP Server로 문서 검색, Claude가 필요 시 호출

---

## 9. reflexion 설계 개선 제안 요약

### 즉시 적용 가능 (Phase 1-2 구현 시)

| # | 패턴 | 출처 | 변경 범위 | 효과 |
|---|------|------|----------|------|
| 1 | **Privacy 태그 스트리핑** | claude-mem | prompt-logger.mjs | 민감 프롬프트 보호 |
| 2 | **Content-Addressable 분석 캐시** | QMD | analysis_cache 스키마 | 동일 입력 재분석 방지 |
| 3 | **Smart Install 체크** | claude-mem | 설치 스크립트 | 첫 실행 UX 개선 |
| 4 | **FTS5 trigger 동기화** | QMD | db.mjs 스키마 | events 텍스트 검색 지원 |

### Phase 5 확장 시

| # | 패턴 | 출처 | 변경 범위 | 효과 |
|---|------|------|----------|------|
| 5 | **MCP 검색 인터페이스** | claude-mem + QMD | 신규 MCP Server | Claude가 능동적으로 에러 KB/분석 결과 검색 |
| 6 | **Progressive Disclosure** | claude-mem | MCP 검색 응답 설계 | 토큰 효율적 검색 결과 제공 |
| 7 | **FTS5 + 벡터 하이브리드** | QMD | error-kb.mjs | 에러 KB 검색 품질 향상 |
| 8 | **Strong-signal shortcut** | QMD | error-kb.mjs | 정확 매칭 시 벡터 검색 스킵 → 속도 개선 |

### 장기 검토

| # | 패턴 | 출처 | 비고 |
|---|------|------|------|
| 9 | **Worker 통합 패턴** | claude-mem | 훅 수가 늘어나면 Worker로 통합 검토 |
| 10 | **Web Viewer UI** | claude-mem | 분석 결과/피드백 시각화 도구 |
| 11 | **Plugin Marketplace 배포** | claude-mem | 설치 편의성 극대화 |

---

## 10. 결론

### 3개 시스템의 위치

- **reflexion**은 "**학습하는 시스템**" — 패턴을 발견하고 자동화를 제안
- **claude-mem**은 "**기억하는 시스템**" — 세션 간 컨텍스트를 보존
- **QMD**는 "**검색하는 시스템**" — 문서에서 정보를 찾음

### 종속성 도입은 부적합

3개 시스템은 각각 다른 문제를 해결하며, 데이터 형태와 처리 방식이 근본적으로 다르다. 종속성으로 도입하면 오히려 복잡성이 증가한다.

### 패턴 차용은 적극 권장

claude-mem의 **Privacy 태그, Progressive Disclosure, MCP 인터페이스**와 QMD의 **Content-Addressable Storage, FTS5 하이브리드, Strong-signal shortcut**은 reflexion의 설계를 즉시 개선할 수 있는 검증된 패턴이다.

### 최적의 관계: 독립 운영 + 자연스러운 합류

3개 시스템이 Claude Code 세션에서 각자의 역할을 수행하며 시너지를 낸다:
1. reflexion이 "이 프롬프트 패턴을 스킬로 만들까요?"라고 제안하고
2. claude-mem이 "지난 세션에서 이 기능을 작업했었습니다"라고 기억을 제공하며
3. QMD가 "관련 문서는 DESIGN.md 3.2절에 있습니다"라고 검색 결과를 보여준다

---

*이 문서는 reflexion 설계 참고용으로 작성되었으며, 각 프로젝트의 실제 구현은 변경될 수 있다.*
