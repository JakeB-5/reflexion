---
id: pre-tool-guide
title: "사전 예방 가이드 (Pre-Tool Guide)"
status: draft
created: 2026-02-07
domain: realtime-assist
depends: "data-collection/log-writer, realtime-assist/error-kb"
constitution_version: "2.0.0"
---

# pre-tool-guide

> `hooks/pre-tool-guide.mjs` — PreToolUse 훅 이벤트를 처리하여 Edit/Write, Bash 도구 실행 전에 과거 에러 이력 기반의 사전 경고/가이드를 Claude에게 주입하는 훅 스크립트. 모든 데이터 조회는 벡터 검색 없이 텍스트 매칭만 사용한다 (성능 우선).

---

## 개요

pre-tool-guide는 도구 실행 전에 과거 학습 데이터를 기반으로 사전 경고를 제공한다. 이를 통해 Claude가 이전에 발생한 에러를 반복하지 않고, 최적의 도구 사용 전략을 선택하도록 유도한다.

> **설계 원칙**: PreToolUse는 동기 블로킹 훅이므로, 벡터 검색 대신 텍스트 매칭만 사용하여 성능을 우선한다. error_kb 테이블을 직접 SQL 쿼리한다.

### 파일 위치

- 훅 스크립트: `~/.reflexion/hooks/pre-tool-guide.mjs`
- 의존 데이터: `~/.reflexion/data/reflexion.db` (`events` 테이블, `error_kb` 테이블)

### 훅 등록

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Edit|Write|Bash|Task",
      "hooks": [{
        "type": "command",
        "command": "node $HOME/.reflexion/hooks/pre-tool-guide.mjs"
      }]
    }]
  }
}
```

> **matcher**: `"Edit|Write|Bash|Task"` — 이 4개 도구에 대해서만 훅이 실행된다. 다른 도구(Read, Grep 등)는 matcher에 의해 필터링되어 훅이 호출되지 않는다.

### 대상 도구별 가이드

| 도구 | 가이드 내용 | 데이터 소스 |
|------|------------|------------|
| Edit | 대상 파일명 관련 error_kb 해결 이력 | `error_kb` 테이블 직접 LIKE 쿼리 |
| Write | 대상 파일명 관련 error_kb 해결 이력 | `error_kb` 테이블 직접 LIKE 쿼리 |
| Bash | 현재 세션 내 Bash 에러 → error_kb 정확 매치 | `events` + `error_kb` 테이블 |
| Task | ~~서브에이전트 실패율 경고~~ (v9: 비활성화) | — |

---

## 요구사항

### REQ-RA-301: Edit/Write 도구 — 파일 관련 에러 이력 주입

시스템은 Edit 또는 Write 도구 실행 전에, 대상 파일과 관련된 과거 에러 이력을 `error_kb` 테이블에서 직접 검색하여 Claude에게 주입해야 한다(SHALL).

> **설계 근거**: PreToolUse는 동기 블로킹 훅이므로 벡터 검색 대신 텍스트 매칭만 사용 (성능 우선).

1. `tool_input.file_path`에서 파일명을 추출한다 (`filePath.split('/').pop()`) (SHALL)
2. `error_kb` 테이블에서 `error_normalized`에 파일명이 포함된 해결 이력을 직접 SQL 쿼리로 최근 2건 조회 (SHALL):
   ```sql
   SELECT error_normalized, resolution FROM error_kb
   WHERE error_normalized LIKE ? AND resolution IS NOT NULL
   ORDER BY last_used DESC LIMIT 2
   ```
   (`?`에는 `%${fileName}%` 바인딩)
3. 조회된 각 결과에 대해 `resolution` JSON을 파싱하여 에러 패턴과 해결 방법을 `additionalContext`에 포함 (SHALL):
   - `⚠️ 이 파일 관련 과거 에러: ${kb.error_normalized}`
   - `   해결 방법: ${res.resolvedBy} (${res.tool})`
   - `   해결 경로: ${res.toolSequence.join(' → ')}` (toolSequence가 있는 경우)
4. `resolution` JSON 파싱 실패 시 원본 문자열을 그대로 표시 (SHALL)
5. 매치가 없으면 Edit/Write 관련 가이드를 출력하지 않는다(SHALL)

#### Scenario RA-301-1: 파일 관련 에러 이력이 error_kb에 존재할 때

- **GIVEN** `error_kb` 테이블에 `error_normalized LIKE '%index.ts%'`인 해결 이력이 있을 때
- **WHEN** `Edit` 도구가 `file_path: "/project/src/index.ts"`로 실행되기 전
- **THEN** `additionalContext`에 "⚠️ 이 파일 관련 과거 에러: ..."과 "해결 방법: ..."이 포함된 JSON을 stdout으로 출력한다

#### Scenario RA-301-2: 파일 관련 에러 이력이 없을 때

- **GIVEN** `error_kb` 테이블에 해당 파일명 관련 해결 이력이 없을 때
- **WHEN** `Write` 도구가 실행되기 전
- **THEN** Edit/Write 관련 가이드는 출력하지 않는다

---

### REQ-RA-302: Bash 도구 — 세션 내 에러 이력 주입

시스템은 Bash 도구 실행 전에, 현재 세션에서 발생한 최근 Bash 에러를 검색하고 error_kb에서 정확 매치로 해결 이력을 조회하여 Claude에게 주입해야 한다(SHALL).

> **설계 근거**: 벡터 검색 불필요. events 테이블에서 세션 내 에러를 찾고, error_kb에서 정확 텍스트 매치만 수행.

1. `events` 테이블에서 현재 `session_id`의 가장 최근 Bash `tool_error` 에러 메시지를 조회 (SHALL):
   ```sql
   SELECT json_extract(data, '$.error') AS error FROM events
   WHERE type = 'tool_error' AND session_id = ? AND json_extract(data, '$.tool') = 'Bash'
   ORDER BY ts DESC LIMIT 1
   ```
2. 에러가 있으면 `error_kb`에서 정확 텍스트 매치로 해결 이력을 조회 (SHALL):
   ```sql
   SELECT error_normalized, resolution FROM error_kb
   WHERE error_normalized = ? AND resolution IS NOT NULL
   LIMIT 1
   ```
3. KB 매치가 있으면 가이드를 `additionalContext`에 포함 (SHALL):
   - `💡 이 세션에서 Bash 에러 발생 이력: ${kbResult.error_normalized}`
   - `   이전 해결 경로: ${resolution.toolSequence.join(' → ')}` (toolSequence가 있는 경우)
4. 에러가 없거나 KB 매치가 없으면 Bash 관련 가이드를 출력하지 않는다(SHALL)

#### Scenario RA-302-1: 세션 내 Bash 에러가 존재하고 KB 정확 매치됨

- **GIVEN** 현재 세션에서 Bash 에러가 발생했고, `error_kb`에 동일 `error_normalized`의 해결 이력이 있을 때
- **WHEN** Bash 도구가 실행되기 전
- **THEN** `additionalContext`에 "💡 이 세션에서 Bash 에러 발생 이력: ..."과 "이전 해결 경로: ..."가 포함된다

#### Scenario RA-302-2: 세션 내 Bash 에러가 없을 때

- **GIVEN** 현재 세션에서 Bash 에러가 없을 때
- **WHEN** Bash 도구가 실행되기 전
- **THEN** Bash 관련 가이드는 출력하지 않는다

#### Scenario RA-302-3: Bash 에러는 있으나 KB 매치 없음

- **GIVEN** 현재 세션에서 Bash 에러가 발생했지만, `error_kb`에 해당 에러의 해결 이력이 없을 때
- **WHEN** Bash 도구가 실행되기 전
- **THEN** Bash 관련 가이드는 출력하지 않는다

---

### REQ-RA-303: Task 도구 — 서브에이전트 실패율 경고 (v9: 비활성화)

> **v9 비활성화**: SubagentStop API가 `error`/`success` 정보를 제공하지 않아, `success` 필드가 항상 `true`로 기록되던 문제가 있었다. 이에 따라 `success` 필드가 `subagent_stop` 이벤트에서 제거되었고 (subagent-tracker.mjs 참조), 이 기능은 비활성화되었다.

이 기능은 현재 비활성화 상태이며, 향후 SubagentStop API에 실패 정보가 추가되면 `agent_transcript_path` 파싱으로 정확한 실패 판정을 구현한 후 복원할 수 있다(MAY).

```javascript
// v9: Disabled — SubagentStop API does not provide error/success information.
// The success field was removed from subagent_stop events (see subagent-tracker.mjs).
// To re-enable, implement agent_transcript_path parsing for failure detection,
// then query the resulting field here.
// if (input.tool_name === 'Task' && input.tool_input?.subagent_type) { ... }
```

#### Scenario RA-303-1: Task 도구 호출 시 (비활성화 상태)

- **GIVEN** PreToolUse 이벤트의 `tool_name`이 `"Task"`
- **WHEN** pre-tool-guide.mjs가 실행되면
- **THEN** 서브에이전트 실패율 경고를 출력하지 않는다 (기능 비활성화)

---

### REQ-RA-304: 출력 형식

시스템은 가이드 항목이 있을 때만 stdout으로 JSON을 출력해야 한다(SHALL).

1. 여러 가이드 항목을 개행(`\n`)으로 결합하여 `additionalContext`에 포함 (SHALL)
2. 가이드 항목이 없으면 stdout 출력 없이 exit 0으로 종료 (SHALL)
3. 출력 형식 (SHALL):
   ```json
   {
     "hookSpecificOutput": {
       "hookEventName": "PreToolUse",
       "additionalContext": "가이드 항목들..."
     }
   }
   ```

#### Scenario RA-304-1: 여러 가이드가 동시에 존재할 때

- **GIVEN** Edit 도구 실행 시 파일 관련 에러 이력이 2건 존재
- **WHEN** pre-tool-guide.mjs가 실행되면
- **THEN** 2건의 가이드를 개행으로 결합하여 단일 `additionalContext`로 출력한다

#### Scenario RA-304-2: 가이드 항목이 없을 때

- **GIVEN** 어떤 도구에 대해서도 관련 에러 이력이 없을 때
- **WHEN** pre-tool-guide.mjs가 실행되면
- **THEN** stdout 출력 없이 exit 0으로 종료한다

---

### REQ-RA-305: 비차단 실행 보장

훅 스크립트는 어떤 상황에서도 exit 0으로 종료해야 한다(SHALL).

1. 전체 로직을 try-catch로 감싸야 한다(SHALL)
2. 예외 발생 시 `process.exit(0)`으로 종료 (SHALL)

#### Scenario RA-305-1: DB 접근 실패 시

- **GIVEN** `reflexion.db` 파일이 손상되어 접근 불가
- **WHEN** pre-tool-guide.mjs가 실행되면
- **THEN** exit code 0으로 정상 종료한다

---

## 비기능 요구사항

### 성능

- 훅 실행 시간은 2초 이내여야 한다(SHALL)
- 벡터 검색을 사용하지 않고 텍스트 매칭만 사용하여 동기 블로킹 훅의 지연을 최소화한다(SHALL)
- SQL 쿼리는 인덱스를 활용하여 대용량 데이터에서도 빠르게 동작해야 한다(SHOULD)

### 출력 형식

- stdout 출력은 `{ "hookSpecificOutput": { "hookEventName": "PreToolUse", "additionalContext": "..." } }` JSON 형태 (SHALL)
- `additionalContext` 문자열에 여러 가이드 항목을 개행(`\n`)으로 구분 (SHOULD)

---

## 제약사항

- `better-sqlite3` 패키지를 사용하여 SQLite 접근 (SHALL)
- `db.mjs`의 `queryEvents()`, `getDb()`, `readStdin()`, `isEnabled()` 함수를 import하여 사용 (SHALL)
- `searchErrorKB()`는 사용하지 않음 — 벡터 검색 회피를 위해 error_kb 테이블 직접 쿼리 (SHALL)
- PreToolUse는 동기 블로킹 훅이므로 async 작업(임베딩 생성 등)을 수행하지 않는다(SHALL)

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| PreToolUse | Claude Code에서 도구 실행 직전에 발생하는 훅 이벤트 |
| matcher | 훅 등록 시 대상 도구를 지정하는 정규표현식 패턴 |
| additionalContext | 훅이 Claude에게 주입하는 컨텍스트 문자열 |
| tool_sequence | 에러 해결 시 사용된 도구의 순서 목록 |
| 동기 블로킹 훅 | 도구 실행 전 동기적으로 실행되어 완료까지 대기하는 훅 유형 |
| 텍스트 매칭 | 벡터 검색 없이 SQL LIKE/= 연산자로 수행하는 문자열 매칭 |
