---
id: realtime-assist
created: 2026-02-07
---

# 도메인: realtime-assist

> Phase 5: 실시간 어시스턴스 레이어

---

## 개요

### 범위 및 책임

이 도메인은 다음을 담당합니다:

- 세션 내 실시간 에러 KB 벡터 검색 및 해결 이력 제공
- 스킬-프롬프트 벡터 유사도 매칭으로 자동 스킬 감지
- 서브에이전트 성능 추적 및 컨텍스트 주입
- PreToolUse 사전 예방 가이드 제공

### 경계 정의

- **포함**: 에러 KB 검색(`error-kb.mjs`), 스킬 매칭(`skill-matcher.mjs`), 서브에이전트 추적, 세션 컨텍스트 주입, PreToolUse 가이드, 임베딩 데몬(`embedding-server.mjs`, `embedding-client.mjs`), 배치 임베딩(`batch-embeddings.mjs`)
- **제외**: 데이터 수집(data-collection), AI 분석(ai-analysis), 제안 적용(suggestion-engine)

---

## 의존성

- `data-collection` — DB 유틸리티(`db.mjs`), 이벤트 데이터
- `ai-analysis` — 분석 결과(패턴) 소비

---

## 스펙 목록

이 도메인에 속한 기능 명세:

| 스펙 ID | 상태 | 설명 |
|---------|------|------|
| error-kb | draft | 에러 해결 이력 KB (3단계 검색: 정확→접두사→벡터) |
| skill-matcher | draft | 프롬프트-스킬 벡터 유사도 + 키워드 하이브리드 매칭 |
| subagent-tracker | draft | SubagentStop 이벤트 기록 훅 |
| subagent-context | draft | SubagentStart 학습 데이터 주입 훅 |
| pre-tool-guide | draft | PreToolUse 사전 예방 가이드 훅 |
| embedding-daemon | draft | 임베딩 데몬 서버/클라이언트 (Unix socket, Transformers.js) |
| batch-embeddings | draft | 배치 임베딩 프로세서 (SessionEnd 후 detached 실행) |

---

## 공개 인터페이스

다른 도메인에서 사용할 수 있는 공개 API/인터페이스:

### 제공하는 기능

- `searchErrorKB(normalizedError, embedding)` — 에러 KB 3단계 검색 (정확→접두사→벡터)
- `recordResolution(errorHash, resolution)` — 에러 해결 이력 UPSERT 기록
- `matchSkill(prompt, embedding)` — 프롬프트에 매칭되는 스킬 검색
- `embedViaServer(texts)` — 임베딩 데몬 클라이언트 (auto-start 포함)
- `isServerRunning()` — 임베딩 데몬 상태 확인
- `startServer()` — 임베딩 데몬 시작

### 제공하는 타입/인터페이스

- `ErrorKBEntry` — 에러 KB 엔트리 스키마 (벡터 임베딩 포함)
- `SkillEmbedding` — 스킬 임베딩 스키마 (384차원)

---

## 비고

배치 분석(Phase 2)과 상호 보완 관계: 배치는 장기 패턴, 실시간은 즉각 대응. 에러 KB와 스킬 매칭은 sqlite-vec 384차원 벡터 유사도 검색을 사용한다.
