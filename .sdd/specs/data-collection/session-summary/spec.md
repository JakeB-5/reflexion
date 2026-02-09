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

> SessionEnd 훅 스크립트 (`hooks/session-summary.mjs`). 세션 종료 시 해당 세션의 이벤트를 SQL 집계하여 요약 이벤트를 기록하고, 조건 충족 시 비동기 AI 분석을 트리거하며, 확률적으로 오래된 이벤트를 정리하고, 배치 임베딩 생성을 detached 프로세스로 수행한다. 최종 구현은 DESIGN.md 5.4절의 확장 버전이다.

---

## Requirement: REQ-DC-401 — 세션 요약 집계

시스템은 SessionEnd 이벤트 발생 시 해당 세션의 모든 이벤트를 SQL 집계하여 SessionSummaryEntry를 기록(SHALL)해야 한다.

### Scenario: 정상 세션 요약 생성

- **GIVEN** 세션 중 프롬프트 5개, 도구 사용 20개, 에러 2개가 기록된 상태
- **WHEN** SessionEnd 훅이 트리거되면
- **THEN** `insertEvent()`로 다음 필드를 포함하는 이벤트가 기록(SHALL)된다: `v: 1`, `type: 'session_summary'`, `ts`, `sessionId`, `project`, `projectPath`, `promptCount: 5`, `toolCounts: { Bash: N, Read: N, ... }`, `toolSequence: ['Bash', 'Read', ...]`, `errorCount: 2`, `uniqueErrors: [...]`, `lastPrompts`, `lastEditedFiles`, `reason`

### Scenario: 도구 사용 횟수 집계 (toolCounts)

- **GIVEN** 세션에서 Bash 10회, Read 5회, Edit 3회, Write 2회가 사용된 상태
- **WHEN** 요약이 생성되면
- **THEN** `toolCounts`가 `{ Bash: 10, Read: 5, Edit: 3, Write: 2 }`로 집계(SHALL)된다

### Scenario: 도구 시퀀스 기록 (toolSequence)

- **GIVEN** 세션에서 도구가 시간 순으로 사용된 상태
- **WHEN** 요약이 생성되면
- **THEN** `toolSequence`가 사용 순서대로 도구명 배열로 기록(SHALL)된다

### Scenario: 고유 에러 수집 (uniqueErrors)

- **GIVEN** 세션에서 동일한 정규화 에러가 3번 발생한 상태
- **WHEN** 요약이 생성되면
- **THEN** `uniqueErrors`에는 `[...new Set(errors.map(e => e.error))]`로 중복 제거된 고유 에러 목록만 포함(SHALL)된다

### Scenario: 마지막 프롬프트 기록 (v7 P2)

- **GIVEN** 세션에서 5개의 프롬프트가 수집된 상태
- **WHEN** 요약이 생성되면
- **THEN** `lastPrompts`에 마지막 3개 프롬프트의 처음 100자가 배열로 포함(SHALL)된다 (`prompts.slice(-3).map(p => (p.text || '').slice(0, 100))`)

### Scenario: 수정 파일 기록 (v7 P2)

- **GIVEN** 세션에서 Edit, Write 도구로 3개의 파일이 수정된 상태
- **WHEN** 요약이 생성되면
- **THEN** `lastEditedFiles`에 수정 파일 목록이 중복 제거 후 최대 5개까지 포함(SHALL)된다

### Scenario: 종료 사유 기록 (v7 P8)

- **GIVEN** SessionEnd의 `reason`이 `'end_turn'`인 상태
- **WHEN** 요약이 생성되면
- **THEN** `reason`에 `input.reason || 'unknown'`이 기록(SHALL)된다

---

## Requirement: REQ-DC-402 — 세션 이벤트 SQL 집계

시스템은 `queryEvents()`를 사용하여 현재 세션의 이벤트를 효율적으로 조회(SHALL)해야 한다.

### Scenario: 세션 이벤트 조회 및 필터링

- **GIVEN** `events` 테이블에 여러 세션의 이벤트가 혼재된 상태
- **WHEN** 세션 요약이 생성되면
- **THEN** `queryEvents({ sessionId: input.session_id })`로 현재 세션의 이벤트를 조회하고, JavaScript에서 `type`별로 필터링(SHALL)한다

### Scenario: 빈 세션

- **GIVEN** 세션 중 어떤 이벤트도 기록되지 않은 상태 (예: 즉시 종료)
- **WHEN** 요약이 생성되면
- **THEN** `promptCount: 0`, `toolCounts: {}`, `toolSequence: []`, `errorCount: 0`, `uniqueErrors: []`로 기록(SHALL)된다

---

## Requirement: REQ-DC-403 — AI 분석 트리거 조건

시스템은 세션 종료 시 특정 조건을 만족하면 비동기 AI 분석을 트리거(SHOULD)해야 한다. `runAnalysisAsync()`는 `ai-analyzer.mjs`에서 import한다.

### Scenario: 프롬프트 3개 이상이고 정상 종료인 세션

- **GIVEN** 세션의 프롬프트 수가 3개 이상이고, `reason`이 `'clear'`가 아닌 상태
- **WHEN** SessionEnd 훅이 실행되면
- **THEN** `runAnalysisAsync({ days: 7, project, projectPath })`를 호출하여 비동기 AI 분석이 트리거(SHOULD)된다

### Scenario: 프롬프트 3개 미만 세션

- **GIVEN** 세션의 프롬프트 수가 2개 이하인 상태
- **WHEN** SessionEnd 훅이 실행되면
- **THEN** AI 분석을 트리거하지 않는다(SHALL)

### Scenario: clear 이유로 종료된 세션 (v7 P8)

- **GIVEN** SessionEnd의 `reason`이 `'clear'`인 상태
- **WHEN** 훅이 실행되면
- **THEN** `skipAnalysis = true`로 설정하여 AI 분석을 트리거하지 않는다(SHALL) (대화 초기화는 분석 대상이 아님)

---

## Requirement: REQ-DC-404 — Non-blocking 실행 보장

시스템은 집계, AI 트리거, 또는 배치 임베딩 실패 시에도 exit code 0으로 종료(SHALL)해야 한다.

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

시스템은 세션 요약 완료 후 detached 프로세스로 배치 임베딩 생성을 트리거(SHOULD)해야 한다. `batch-embeddings.mjs`를 `child_process.spawn()`으로 실행하며, SessionEnd 훅을 블로킹하지 않는다.

### Scenario: detached 배치 임베딩 프로세스 생성

- **GIVEN** 세션 요약이 정상적으로 기록된 상태
- **WHEN** 세션 요약 처리가 완료되면
- **THEN** `spawn('node', [batchScript, projectPath], { detached: true, stdio: 'ignore' })`로 배치 임베딩 프로세스를 생성하고 `child.unref()`로 분리(SHOULD)한다

### Scenario: 배치 임베딩 트리거 실패

- **GIVEN** `child_process.spawn()` 호출이 실패하는 상태
- **WHEN** 배치 임베딩 생성 중 예외가 발생하면
- **THEN** try-catch로 포착하고 세션 요약은 정상 완료된 상태로 exit 0으로 종료(SHALL)한다

---

## Requirement: REQ-DC-406 — 확률적 DB 정리 (v9)

시스템은 세션 요약 처리 시 확률적으로(10%) 오래된 이벤트를 정리(SHOULD)해야 한다.

### Scenario: 확률적 pruning 트리거

- **GIVEN** 세션 요약이 기록된 상태
- **WHEN** `Math.random() < 0.1` 조건이 참이면
- **THEN** `pruneOldEvents()`를 호출하여 보관 기한 초과 이벤트와 미사용 에러 KB를 삭제(SHOULD)한다

### Scenario: pruning 실패

- **GIVEN** `pruneOldEvents()` 실행 중 예외가 발생하는 상태
- **WHEN** 정리 작업이 실패하면
- **THEN** try-catch로 포착하고 정상 진행(SHALL)한다 (non-critical)

---

## Requirement: REQ-DC-407 — 시스템 활성화 체크

시스템은 훅 실행 초기에 `isEnabled()`를 확인하여 비활성화 시 즉시 종료(SHALL)해야 한다.

### Scenario: 시스템 비활성화

- **GIVEN** `config.json`에서 `enabled: false`로 설정된 상태
- **WHEN** 훅이 실행되면
- **THEN** `isEnabled()` 확인 후 즉시 `process.exit(0)`으로 종료(SHALL)한다

---

## 비고

- **5.4절 확장 버전이 최종본**: DESIGN.md 4.6절(Phase 1 기본)이 아닌 5.4절(확장 버전)을 구현한다. Phase 1 기본 버전을 구현하지 말 것
- AI 분석 트리거: `import { runAnalysisAsync } from '../lib/ai-analyzer.mjs'`
- DB 정리: `import { pruneOldEvents } from '../lib/db.mjs'` — session-summary에서 10% 확률로 호출
- `toolSequence`는 워크플로우 패턴 분석에 활용 (예: Read→Edit→Bash 반복 패턴 감지)
- 배치 임베딩 스크립트 경로: `~/.self-generation/lib/batch-embeddings.mjs`
- 배치 임베딩 스크립트는 10초 딜레이 후 실행하여 DB 쓰기 경합을 줄인다 (WAL + busy_timeout으로 최종 보장)
- 저장소가 JSONL에서 SQLite `events` 테이블로 변경됨. `db.mjs`의 `insertEvent()`와 `queryEvents()`를 사용한다
