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

> PostToolUseFailure 훅 스크립트 (`hooks/error-logger.mjs`). 도구 실행 실패를 수집하고, 에러를 정규화하여 `events` 테이블에 기록하며, 에러 KB에서 벡터 유사도 + 텍스트 폴백 검색을 통해 과거 해결 이력을 검색하여 Claude에게 주입한다. v6 확장 버전(DESIGN.md 8.1절 하단)이 최종 구현이다.

---

## Requirement: REQ-DC-301 — 에러 수집

시스템은 PostToolUseFailure 이벤트 발생 시 에러 데이터를 `events` 테이블에 기록(SHALL)해야 한다. 공통 필드는 각 컬럼에, 에러 고유 필드(`tool`, `error`, `errorRaw`)는 `data` JSON 컬럼에 저장된다.

### Scenario: 도구 실행 실패 기록

- **GIVEN** Claude Code 도구 실행이 실패한 상태
- **WHEN** PostToolUseFailure 훅이 트리거되면
- **THEN** `insertEvent()`로 다음 필드를 포함하는 이벤트가 기록(SHALL)된다: `v: 1`, `type: 'tool_error'`, `ts`, `sessionId`, `project`, `projectPath`, `tool`, `error` (정규화), `errorRaw` (원본 500자)

### Scenario: stdin 필드 매핑

- **GIVEN** Claude Code가 stdin으로 `{ tool_name, tool_input, error, session_id, cwd }`를 전달하는 상태
- **WHEN** 훅이 실행되면
- **THEN** `normalizeError(input.error || '')` → `error`, `(input.error || '').slice(0, 500)` → `errorRaw`, `input.tool_name` → `tool`로 매핑(SHALL)된다

---

## Requirement: REQ-DC-302 — 에러 정규화 (normalizeError)

시스템은 `error-kb.mjs`에서 import한 `normalizeError()`를 사용하여 에러 메시지를 정규화(SHALL)해야 한다. 정규화 규칙의 상세 정의는 `realtime-assist/error-kb` 스펙의 REQ-RA-001을 참조한다.

### Scenario: 정규화 함수 사용

- **GIVEN** v6 확장 구현 상태
- **WHEN** error-logger.mjs가 에러를 정규화해야 하면
- **THEN** `error-kb.mjs`에서 `normalizeError()`를 import하여 사용(SHALL)한다 (단일 소유자 원칙)

### Scenario: 정규화 규칙 요약

- **GIVEN** 에러 메시지 `"Cannot find module '/Users/user/src/index.ts'"`가 입력된 상태
- **WHEN** `normalizeError(error)`가 호출되면
- **THEN** 경로→`<PATH>`, 숫자→`<N>`, 문자열→`<STR>` 치환 후 200자 절단된 결과를 반환(SHALL)한다

---

## Requirement: REQ-DC-303 — 에러 KB 실시간 검색 및 컨텍스트 주입

시스템은 에러 발생 시 에러 KB에서 과거 해결 이력을 검색(SHALL)하고, 매칭 결과가 있으면 `additionalContext`로 Claude에게 주입해야 한다. 검색은 2초 타임아웃으로 제한하여 데몬 콜드 스타트 시 블로킹을 방지한다.

### Scenario: 과거 해결 이력 발견

- **GIVEN** 에러 KB에 동일/유사 정규화 에러에 대한 해결 기록이 존재하는 상태
- **WHEN** `searchErrorKB(normalizedError)`가 매칭 결과를 반환하면
- **THEN** stdout으로 다음 형식의 JSON을 출력(SHALL)한다:
  ```json
  {
    "hookSpecificOutput": {
      "hookEventName": "PostToolUseFailure",
      "additionalContext": "[Reflexion 에러 KB] 이전에 동일 에러를 해결한 이력이 있습니다:\n- 에러: <error_normalized>\n- 해결 방법: <resolution>\n이 정보를 참고하여 해결을 시도하세요."
    }
  }
  ```

### Scenario: resolution JSON 파싱

- **GIVEN** `kbMatch.resolution`이 JSON 문자열인 상태
- **WHEN** 컨텍스트를 구성할 때
- **THEN** `JSON.parse()`로 파싱하여 `resolvedBy`와 `toolSequence`를 추출(SHOULD)하고, 파싱 실패 시 원본 문자열을 사용한다

### Scenario: 2초 타임아웃 적용 (v9)

- **GIVEN** 임베딩 데몬이 콜드 스타트 중인 상태
- **WHEN** `searchErrorKB()` 호출이 2초를 초과하면
- **THEN** `Promise.race()`로 타임아웃하여 `null`을 반환하고 컨텍스트 주입 없이 종료(SHALL)한다

### Scenario: 해결 이력 미발견

- **GIVEN** 에러 KB에 매칭되는 해결 기록이 없는 상태
- **WHEN** `searchErrorKB(normalizedError)`가 `null`을 반환하면
- **THEN** stdout 출력 없이 정상 종료(SHALL)한다

---

## Requirement: REQ-DC-304 — Non-blocking 실행 보장

시스템은 에러 KB 검색 실패를 포함한 모든 오류 상황에서 exit code 0으로 종료(SHALL)해야 한다.

### Scenario: KB 검색 중 오류

- **GIVEN** 에러 KB DB 조회가 실패하는 상태
- **WHEN** `searchErrorKB()` 호출 중 예외가 발생하면
- **THEN** 에러 기록은 정상 완료하고 KB 검색은 건너뛴 후 exit 0으로 종료(SHALL)한다

### Scenario: 전체 훅 오류

- **GIVEN** stdin 파싱 또는 DB 쓰기가 실패하는 상태
- **WHEN** 최외곽 try-catch에서 예외가 포착되면
- **THEN** `process.exit(0)`으로 정상 종료(SHALL)한다

---

## Requirement: REQ-DC-305 — 시스템 활성화 체크

시스템은 훅 실행 초기에 `isEnabled()`를 확인하여 비활성화 시 즉시 종료(SHALL)해야 한다.

### Scenario: 시스템 비활성화

- **GIVEN** `config.json`에서 `enabled: false`로 설정된 상태
- **WHEN** 훅이 실행되면
- **THEN** `isEnabled()` 확인 후 즉시 `process.exit(0)`으로 종료(SHALL)한다

---

## 비고

- **v6 확장 버전이 최종본**: DESIGN.md 4.5절(Phase 1 기본)이 아닌 8.1절 하단(v6 확장)을 구현한다. Phase 1과 v6를 병합하지 말 것
- `normalizeError()`는 `error-kb.mjs`에서 import (단일 소유자 원칙). `import { normalizeError, searchErrorKB } from '../lib/error-kb.mjs'`
- `searchErrorKB()`의 3단계 검색: (1) 정확한 텍스트 매칭 ~1ms, (2) 접두사 매칭 + 길이비율 70% ~2ms, (3) 벡터 유사도 검색 distance < 0.76 ~5ms
- `errorRaw`는 원본 에러의 처음 500자를 저장하여 디버깅 시 참고용으로 활용
- 정규화 순서가 중요: 경로 → 숫자 → 문자열 (경로 내 숫자가 먼저 `<N>`으로 치환되는 것을 방지)
- 저장소가 JSONL에서 SQLite `events` 테이블로 변경됨. `db.mjs`의 `insertEvent()`를 사용하여 이벤트를 기록한다
