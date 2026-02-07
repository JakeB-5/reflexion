---
id: error-logger
title: "error-logger"
status: draft
created: 2026-02-07
domain: data-collection
depends: "data-collection/log-writer, realtime-assist/error-kb"
constitution_version: "2.0.0"
---

# error-logger

> PostToolUseFailure 훅 스크립트 (`hooks/error-logger.mjs`). 도구 실행 실패를 수집하고, 에러를 정규화하여 `events` 테이블에 기록하며, 에러 KB에서 벡터 검색을 통해 과거 해결 이력을 검색하여 Claude에게 주입한다.

---

## Requirement: REQ-DC-301 — 에러 수집

시스템은 PostToolUseFailure 이벤트 발생 시 에러 데이터를 `events` 테이블에 기록(SHALL)해야 한다. 공통 필드는 각 컬럼에, 에러 고유 필드(`tool`, `error`, `errorRaw`)는 `data` JSON 컬럼에 저장된다.

### Scenario: 도구 실행 실패 기록

- **GIVEN** Claude Code 도구 실행이 실패한 상태
- **WHEN** PostToolUseFailure 훅이 트리거되면
- **THEN** `insertEvent()`로 다음 필드를 포함하는 이벤트가 기록(SHALL)된다: `v: 1`, `type: 'tool_error'`, `ts`, `session_id`, `project`, `project_path`, `data: { tool, error, errorRaw }`

### Scenario: stdin 필드 매핑

- **GIVEN** Claude Code가 stdin으로 `{ tool_name, tool_input, error, session_id, cwd }`를 전달하는 상태
- **WHEN** 훅이 실행되면
- **THEN** `input.error` → `normalizeError()` → `data.error`, `input.error.slice(0, 500)` → `data.errorRaw`로 매핑(SHALL)된다

---

## Requirement: REQ-DC-302 — 에러 정규화 (normalizeError)

시스템은 에러 메시지를 정규화(SHALL)하여 동일 유형의 에러를 패턴 매칭할 수 있게 해야 한다. 정규화 규칙의 상세 정의는 `realtime-assist/error-kb` 스펙의 REQ-RA-001을 참조한다.

### Scenario: Phase 1 인라인 구현

- **GIVEN** Phase 1 구현에서 `error-kb.mjs` 모듈이 아직 없는 상태
- **WHEN** error-logger.mjs가 에러를 정규화해야 하면
- **THEN** 훅 스크립트 내 인라인 `normalizeError()` 함수를 사용하여 경로→`<PATH>`, 숫자→`<N>`, 문자열→`<STR>` 치환 후 200자 절단(SHALL)한다

### Scenario: Phase 5 import 전환

- **GIVEN** Phase 5에서 `error-kb.mjs` 모듈이 사용 가능한 상태
- **WHEN** error-logger.mjs가 에러를 정규화해야 하면
- **THEN** `error-kb.mjs`에서 `normalizeError()`를 import하여 사용(SHALL)한다

---

## Requirement: REQ-DC-303 — 에러 KB 실시간 검색 및 컨텍스트 주입

시스템은 에러 발생 시 에러 KB에서 과거 해결 이력을 검색(SHALL)하고, 매칭 결과가 있으면 `additionalContext`로 Claude에게 주입해야 한다. Phase 5에서는 벡터 유사도 검색(`vectorSearch`)을 통해 의미적으로 유사한 에러를 매칭한다.

### Scenario: 과거 해결 이력 발견

- **GIVEN** 에러 KB에 동일한 정규화 에러에 대한 해결 기록이 존재하는 상태
- **WHEN** `searchErrorKB(normalizedError)`가 매칭 결과를 반환하면
- **THEN** stdout으로 `{ hookSpecificOutput: { hookEventName: 'PostToolUseFailure', additionalContext: '...' } }` JSON을 출력(SHALL)한다

### Scenario: 해결 이력 미발견

- **GIVEN** 에러 KB에 매칭되는 해결 기록이 없는 상태
- **WHEN** `searchErrorKB(normalizedError)`가 `null`을 반환하면
- **THEN** stdout 출력 없이 정상 종료(SHALL)한다

### Scenario: additionalContext 메시지 형식

- **GIVEN** KB에서 해결 이력이 발견된 상태
- **WHEN** 컨텍스트를 주입할 때
- **THEN** 메시지에 에러 패턴과 해결 방법이 포함(SHALL)되어야 한다

---

## Requirement: REQ-DC-304 — Non-blocking 실행 보장

시스템은 에러 KB 검색 실패를 포함한 모든 오류 상황에서 exit code 0으로 종료(SHALL)해야 한다.

### Scenario: KB 검색 중 오류

- **GIVEN** 에러 KB DB 조회가 실패하는 상태
- **WHEN** `searchErrorKB()` 호출 중 예외가 발생하면
- **THEN** 에러 기록은 정상 완료하고 KB 검색은 건너뛴 후 exit 0으로 종료(SHALL)한다

---

## 비고

- Phase 1 기본 버전에서는 `normalizeError`가 훅 스크립트 내에 인라인으로 구현됨
- Phase 5 확장 시 `error-kb.mjs`에서 `searchErrorKB`를 import하여 실시간 검색 기능이 활성화됨. 내부적으로 `vectorSearch()`를 통한 의미적 유사도 검색을 사용한다
- `errorRaw`는 원본 에러의 처음 500자를 저장하여 디버깅 시 참고용으로 활용
- 정규화 순서가 중요: 경로 → 숫자 → 문자열 (경로 내 숫자가 먼저 `<N>`으로 치환되는 것을 방지)
- 저장소가 JSONL에서 SQLite `events` 테이블로 변경됨. `db.mjs`의 `insertEvent()`를 사용하여 이벤트를 기록한다

### normalizeError 소유권

DESIGN.md 코드상 error-logger.mjs에 인라인 normalizeError가 존재하나, Phase 5 완성 시 `realtime-assist/error-kb` 모듈의 normalizeError를 import하여 사용하는 것이 목표. 구현 시 Phase에 따라 인라인(Phase 1) 또는 import(Phase 5) 중 택일.
