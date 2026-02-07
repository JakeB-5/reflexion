---
id: session-start-hook
title: "SessionStart 캐시 주입 훅"
status: draft
created: 2026-02-07
domain: ai-analysis
depends: "ai-analysis/ai-analyzer, data-collection/log-writer"
constitution_version: "2.0.0"
---

# SessionStart 캐시 주입 훅 (session-start-hook)

> SessionStart 이벤트에서 `analysis_cache` 테이블의 캐시된 AI 분석 결과 제안과 `events` 테이블의 이전 세션 컨텍스트를 `additionalContext` stdout으로 주입하는 훅 스크립트 (`hooks/session-analyzer.mjs`). DB 접근은 `lib/db.mjs`를 통한다.

---

## 요구사항

### REQ-AA-201: 캐시된 분석 결과 주입

시스템은 SessionStart 시 `getCachedAnalysis(24)`를 호출하여 24시간 이내의 캐시된 분석 결과를 조회하고, 제안이 존재하면 최대 3개를 `additionalContext`로 주입해야 한다(SHALL).

#### Scenario: 유효한 캐시에서 제안 주입

- **GIVEN** `analysis_cache` 테이블에 2시간 전 분석 결과가 저장되어 있고, suggestions 5개가 포함되어 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** stdout으로 `{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "<메시지>" } }` JSON이 출력되며, `additionalContext`에는 상위 3개 제안의 type, summary, id가 포함된다

#### Scenario: 캐시 없음 시 무출력 종료

- **GIVEN** `analysis_cache` 테이블이 비어 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** stdout에 아무것도 출력되지 않고 exit code 0으로 종료된다

#### Scenario: 캐시 만료 시 무출력 종료

- **GIVEN** `analysis_cache` 테이블에 30시간 전 분석 결과만 저장되어 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** stdout에 아무것도 출력되지 않고 exit code 0으로 종료된다

---

### REQ-AA-202: 제안 메시지 포맷팅

시스템은 제안을 사용자에게 표시할 형태로 포맷팅해야 한다(SHALL). 각 제안은 `[type] summary [id: suggest-N]` 형식이어야 하며(SHALL), 적용/거부 CLI 명령어 안내를 포함해야 한다(SHALL).

#### Scenario: 제안 포맷팅

- **GIVEN** suggestions 배열에 `{ type: 'skill', summary: 'TS 프로젝트 초기화 스킬', id: 'suggest-0' }` 항목이 있다
- **WHEN** `formatSuggestionsForContext(suggestions)`를 호출한다
- **THEN** 출력 문자열에 `'[Self-Generation] AI 패턴 분석 결과:'` 헤더, `'- [skill] TS 프로젝트 초기화 스킬 [id: suggest-0]'` 제안 항목, `'node ~/.self-generation/bin/apply.mjs'` 적용 안내, `'node ~/.self-generation/bin/dismiss.mjs'` 거부 안내가 포함된다

---

### REQ-AA-203: 이전 세션 컨텍스트 연속성

시스템은 `events` 테이블에서 마지막 `session_summary` 타입 레코드를 조회하여 이전 세션의 컨텍스트(마지막 프롬프트, 편집 파일, 미해결 에러)를 제공해야 한다(SHOULD).

```sql
SELECT * FROM events WHERE type = 'session_summary' ORDER BY ts DESC LIMIT 1
```

#### Scenario: 이전 세션 요약 주입

- **GIVEN** `events` 테이블에 이전 세션의 `type = 'session_summary'` 레코드가 존재하고, `lastPrompts`와 `lastEditedFiles` 필드가 포함되어 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `additionalContext`에 이전 세션의 마지막 프롬프트와 편집 파일 정보가 포함된다

#### Scenario: 이전 세션 요약 없음

- **GIVEN** `events` 테이블에 `type = 'session_summary'` 레코드가 없다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** 이전 세션 컨텍스트 없이 제안만 주입되거나 (제안도 없으면) 무출력 종료된다

---

### REQ-AA-204: 세션 재개 감지 (Resume)

시스템은 stdin의 `source` 필드가 `'resume'`인 경우 세션 재개로 판단하고, 미해결 에러 정보를 `[RESUME]` 태그와 함께 주입해야 한다(SHOULD).

#### Scenario: 재개 세션에서 미해결 에러 주입

- **GIVEN** SessionStart stdin에 `source: 'resume'`이 포함되고, 이전 세션에 미해결 에러 2개가 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `additionalContext`에 `'[RESUME]'` 태그와 함께 미해결 에러 정보가 포함된다

#### Scenario: 일반 세션 시작

- **GIVEN** SessionStart stdin에 `source: 'new'`이 포함된다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `[RESUME]` 태그는 포함되지 않고 일반 제안만 주입된다

---

### REQ-AA-205: 훅 실패 안전성

시스템은 어떤 예외가 발생하더라도 exit code 0으로 종료해야 한다(SHALL). SessionStart 훅 실패가 Claude Code 세션 시작을 방해해서는 안 된다(SHALL NOT).

#### Scenario: DB 손상 또는 접근 실패

- **GIVEN** `self-gen.db` 파일이 손상되었거나 접근할 수 없다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** 예외가 catch되고 exit code 0으로 종료된다

#### Scenario: db.mjs 모듈 로드 실패

- **GIVEN** `db.mjs` 모듈이 존재하지 않거나 import에 실패한다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** 최상위 try-catch에서 예외가 처리되고 exit code 0으로 종료된다

---

## 비기능 요구사항

### 성능

- SessionStart 훅 전체 실행 시간은 100ms 이내여야 한다(SHALL)
- DB 조회만 수행하며, AI 호출이나 무거운 연산을 수행해서는 안 된다(SHALL NOT)

### 안정성

- 모든 코드 경로에서 exit code 0을 보장해야 한다(SHALL)
- 최상위 try-catch 블록으로 모든 예외를 포착해야 한다(SHALL)

---

## 제약사항

- stdin으로 `{ source, model, session_id, cwd }` 형태의 JSON을 받는다
- stdout JSON 형식: `{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "<문자열>" } }`
- `ai-analysis/ai-analyzer`의 `getCachedAnalysis()`와 `data-collection/db`의 `queryEvents()`에 의존한다
- Claude Code Hooks API 규격을 준수해야 한다

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| `hooks/session-analyzer.mjs` | SessionStart 이벤트 훅 스크립트 |
| `additionalContext` | 훅이 Claude에게 주입하는 컨텍스트 문자열 |
| `source: 'resume'` | 이전 세션을 이어서 시작하는 경우의 stdin 필드 값 |
| `[RESUME]` | 세션 재개 시 미해결 에러에 붙는 태그 |
