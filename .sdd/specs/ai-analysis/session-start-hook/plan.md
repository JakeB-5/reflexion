---
spec: session-start-hook
created: 2026-02-07
domain: ai-analysis
---

# 구현 계획: SessionStart 캐시 주입 훅 (session-start-hook)

## 개요

SessionStart 이벤트에서 캐시된 AI 분석 제안과 이전 세션 컨텍스트를 Claude에게 주입하는 훅 스크립트. 100ms 이내 실행과 절대 실패 안전성을 보장한다.

---

## Phase 1: 기반 구축

### 1.1 훅 스크립트 스캐폴딩
- `~/.reflexion/hooks/session-analyzer.mjs` 파일 생성
- 최상위 try-catch 블록으로 전체 코드 감싸기
- stdin 읽기 (`readStdin()` from log-writer)
- 의존 모듈 import: `ai-analyzer.mjs`, `log-writer.mjs`

---

## Phase 2: 핵심 구현

### 2.1 캐시 조회 및 제안 주입
- `getCachedAnalysis(24)` 호출
- suggestions 존재 여부 확인
- 상위 3개 제안을 `formatSuggestionsForContext()`로 포맷팅

### 2.2 제안 포맷팅 함수
- `[Reflexion] AI 패턴 분석 결과:` 헤더
- 각 제안: `- [type] summary [id: suggest-N]` 형식
- 적용/거부 CLI 안내 푸터

### 2.3 이전 세션 컨텍스트 연속성
- `events` 테이블에서 마지막 `session_summary` 엔트리 조회
- `lastPrompts`, `lastEditedFiles`, `uniqueErrors` 정보 추출
- additionalContext에 이전 세션 컨텍스트 추가

### 2.4 세션 재개 감지
- stdin `source === 'resume'` 확인
- 이전 세션의 미해결 에러에 `[RESUME]` 태그 부여
- additionalContext에 재개 정보 추가

### 2.5 stdout 출력
- `hookSpecificOutput.hookEventName: 'SessionStart'` 형식 JSON 출력
- `process.stdout.write(JSON.stringify(output))`

---

## Phase 3: 테스트 및 검증

### 3.1 단위 테스트
- `formatSuggestionsForContext()`: 포맷 검증, 최대 3개 제한 검증
- 캐시 없음/만료/유효 분기 테스트
- resume 감지 로직 테스트
- 예외 처리: 캐시 파손, 모듈 부재 시 exit 0 보장

### 3.2 통합 테스트
- 실제 캐시 파일 생성 후 훅 실행 → stdout JSON 검증
- 빈 상태에서 훅 실행 → 무출력 + exit 0 검증

---

## 의존성

- `ai-analysis/ai-analyzer`: `getCachedAnalysis()` 함수
- `data-collection/log-writer`: `readStdin()`, `queryEvents()` 함수
