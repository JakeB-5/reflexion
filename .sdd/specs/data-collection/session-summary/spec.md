---
id: session-summary
title: "session-summary"
status: draft
created: 2026-02-07
domain: data-collection
depends: "data-collection/log-writer, ai-analysis/ai-analyzer"
constitution_version: "2.0.0"
---

# session-summary

> SessionEnd 훅 스크립트 (`hooks/session-summary.mjs`). 세션 종료 시 해당 세션의 이벤트를 SQL 집계하여 요약 이벤트를 기록하고, 조건 충족 시 비동기 AI 분석을 트리거하며, 새로운 error_kb 엔트리에 대한 배치 임베딩 생성을 수행한다.

---

## Requirement: REQ-DC-401 — 세션 요약 집계

시스템은 SessionEnd 이벤트 발생 시 해당 세션의 모든 이벤트를 SQL 집계하여 SessionSummaryEntry를 기록(SHALL)해야 한다.

### Scenario: 정상 세션 요약 생성

- **GIVEN** 세션 중 프롬프트 5개, 도구 사용 20개, 에러 2개가 기록된 상태
- **WHEN** SessionEnd 훅이 트리거되면
- **THEN** `insertEvent()`로 다음 필드를 포함하는 이벤트가 기록(SHALL)된다: `v: 1`, `type: 'session_summary'`, `ts`, `session_id`, `project`, `project_path`, `data: { promptCount: 5, toolCounts: { Bash: N, Read: N, ... }, toolSequence: ['Bash', 'Read', ...], errorCount: 2, uniqueErrors: [...], lastPrompts, lastEditedFiles, reason }`

### Scenario: 도구 사용 횟수 집계 (toolCounts)

- **GIVEN** 세션에서 Bash 10회, Read 5회, Edit 3회, Write 2회가 사용된 상태
- **WHEN** 요약이 생성되면
- **THEN** `data.toolCounts`가 `{ Bash: 10, Read: 5, Edit: 3, Write: 2 }`로 집계(SHALL)된다

### Scenario: 도구 시퀀스 기록 (toolSequence)

- **GIVEN** 세션에서 도구가 시간 순으로 사용된 상태
- **WHEN** 요약이 생성되면
- **THEN** `data.toolSequence`가 사용 순서대로 도구명 배열로 기록(SHALL)된다

### Scenario: 고유 에러 수집 (uniqueErrors)

- **GIVEN** 세션에서 동일한 정규화 에러가 3번 발생한 상태
- **WHEN** 요약이 생성되면
- **THEN** `data.uniqueErrors`에는 중복 제거된 고유 에러 목록만 포함(SHALL)된다

### Scenario: 마지막 프롬프트 및 수정 파일 기록

- **GIVEN** 세션에서 5개의 프롬프트가 수집되고, 3개의 파일이 수정되었으며, 종료 사유가 'end_turn'이면
- **WHEN** SessionEnd 훅이 실행되면
- **THEN** lastPrompts에 마지막 3개 프롬프트 요약(100자), lastEditedFiles에 수정 파일 목록(중복 제거, 최대 5개), reason='end_turn'이 포함되어야 한다

---

## Requirement: REQ-DC-402 — 세션 이벤트 SQL 집계

시스템은 SQL 집계 쿼리를 사용하여 현재 세션의 이벤트를 효율적으로 조회(SHALL)해야 한다.

### Scenario: SQL 기반 타입별 집계

- **GIVEN** `events` 테이블에 여러 세션의 이벤트가 혼재된 상태
- **WHEN** 세션 요약이 생성되면
- **THEN** 다음 SQL로 현재 세션의 이벤트를 타입별로 집계(SHALL)한다:
  ```sql
  SELECT type, COUNT(*) as count FROM events WHERE session_id = ? GROUP BY type
  ```

### Scenario: 빈 세션

- **GIVEN** 세션 중 어떤 이벤트도 기록되지 않은 상태 (예: 즉시 종료)
- **WHEN** 요약이 생성되면
- **THEN** `data.promptCount: 0`, `data.toolCounts: {}`, `data.errorCount: 0`으로 기록(SHALL)된다

---

## Requirement: REQ-DC-403 — AI 분석 트리거 조건

시스템은 세션 종료 시 특정 조건을 만족하면 비동기 AI 분석을 트리거(SHOULD)해야 한다.

### Scenario: 프롬프트 3개 이상 세션

- **GIVEN** 세션의 프롬프트 수가 3개 이상인 상태
- **WHEN** SessionEnd 훅이 실행되면
- **THEN** 비동기 AI 분석이 트리거(SHOULD)된다 (`runAnalysisAsync()` 호출)

### Scenario: 프롬프트 3개 미만 세션

- **GIVEN** 세션의 프롬프트 수가 2개 이하인 상태
- **WHEN** SessionEnd 훅이 실행되면
- **THEN** AI 분석을 트리거하지 않는다(SHALL)

### Scenario: clear 이유로 종료된 세션

- **GIVEN** SessionEnd의 `reason`이 `'clear'`인 상태
- **WHEN** 훅이 실행되면
- **THEN** AI 분석을 트리거하지 않는다(SHALL) (대화 초기화는 분석 대상이 아님)

---

## Requirement: REQ-DC-404 — Non-blocking 실행 보장

시스템은 집계 또는 AI 트리거 실패 시에도 exit code 0으로 종료(SHALL)해야 한다.

### Scenario: DB 조회 실패

- **GIVEN** DB 연결 또는 쿼리가 실패하는 상태
- **WHEN** 세션 요약 생성 중 예외가 발생하면
- **THEN** try-catch로 포착하고 `process.exit(0)`으로 정상 종료(SHALL)한다

### Scenario: AI 분석 트리거 실패

- **GIVEN** AI 분석 프로세스 생성이 실패하는 상태
- **WHEN** `runAnalysisAsync()` 호출 중 예외가 발생하면
- **THEN** 세션 요약 기록은 정상 완료하고 exit 0으로 종료(SHALL)한다

---

## Requirement: REQ-DC-405 — 배치 임베딩 생성 트리거

시스템은 세션 요약 완료 후 임베딩이 없는 새로운 `error_kb` 엔트리에 대해 배치 임베딩 생성을 트리거(SHOULD)해야 한다.

### Scenario: 임베딩 미생성 error_kb 엔트리 존재

- **GIVEN** 세션 요약이 정상적으로 기록되고, `error_kb` 테이블에 임베딩이 없는 새 엔트리가 존재하는 상태
- **WHEN** 세션 요약 처리가 완료되면
- **THEN** `generateEmbeddings()`를 호출하여 미생성 엔트리의 임베딩을 배치 생성(SHOULD)한다

### Scenario: 임베딩 생성 실패

- **GIVEN** `generateEmbeddings()` 호출이 실패하는 상태
- **WHEN** 배치 임베딩 생성 중 예외가 발생하면
- **THEN** 예외를 포착하고 세션 요약은 정상 완료된 상태로 exit 0으로 종료(SHALL)한다

---

## 비고

- Phase 1 기본 버전에서는 AI 분석 트리거 및 배치 임베딩 생성 없이 요약 기록만 수행
- Phase 2 확장 시 `ai-analyzer.mjs`에서 `runAnalysisAsync`를 import하여 비동기 분석 활성화
- `toolSequence`는 워크플로우 패턴 분석에 활용 (예: Read→Edit→Bash 반복 패턴 감지)
- SQL 집계 쿼리를 사용하여 세션 이벤트를 효율적으로 처리 (JSONL 전체 파일 스캔 대비 성능 향상)
- 저장소가 JSONL에서 SQLite `events` 테이블로 변경됨. `db.mjs`의 `insertEvent()`와 SQL 집계를 사용한다
