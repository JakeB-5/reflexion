---
id: prompt-logger
title: "prompt-logger"
status: draft
created: 2026-02-07
domain: data-collection
depends: "data-collection/log-writer, realtime-assist/skill-matcher"
constitution_version: "2.0.0"
---

# prompt-logger

> UserPromptSubmit 훅 스크립트 (`hooks/prompt-logger.mjs`). 사용자 프롬프트를 수집하여 `events` 테이블에 기록하고, 슬래시 커맨드(`/`) 사용 시 스킬 사용 이벤트를 별도 기록한다.

---

## Requirement: REQ-DC-101 — 프롬프트 수집

시스템은 UserPromptSubmit 이벤트 발생 시 프롬프트 데이터를 `events` 테이블에 기록(SHALL)해야 한다. 공통 필드는 각 컬럼에, 프롬프트 고유 필드(`text`, `charCount`)는 `data` JSON 컬럼에 저장된다.

### Scenario: 일반 프롬프트 기록

- **GIVEN** 사용자가 Claude Code에 프롬프트를 입력한 상태
- **WHEN** UserPromptSubmit 훅이 트리거되면
- **THEN** 다음 필드를 포함하는 이벤트가 `insertEvent()`로 기록(SHALL)된다: `v: 1`, `type: 'prompt'`, `ts` (ISO 8601), `session_id`, `project` (디렉토리명), `project_path` (전체 경로), `data: { text, charCount }`

### Scenario: stdin 필드 매핑

- **GIVEN** Claude Code가 stdin으로 `{ prompt, session_id, cwd }`를 전달하는 상태
- **WHEN** 훅이 실행되면
- **THEN** `input.session_id` → `session_id`, `input.cwd` → `project_path`, `getProjectName(input.cwd)` → `project`로 매핑(SHALL)된다

---

## Requirement: REQ-DC-102 — 프라이버시 보호 (프롬프트 텍스트 수집 제어)

시스템은 설정에 따라 프롬프트 원문 수집을 비활성화(SHALL)할 수 있어야 한다.

### Scenario: collectPromptText=false 설정 시

- **GIVEN** `config.json`에서 `collectPromptText: false`로 설정된 상태
- **WHEN** 프롬프트가 기록되면
- **THEN** `data.text` 필드가 `"[REDACTED]"`로 대체(SHALL)되고, `data.charCount`는 원본 길이를 유지한다

### Scenario: 기본값 (수집 활성화)

- **GIVEN** `config.json`이 없거나 `collectPromptText`가 명시되지 않은 상태
- **WHEN** 프롬프트가 기록되면
- **THEN** `data.text` 필드에 원문이 그대로 기록(SHALL)된다

---

## Requirement: REQ-DC-103 — 스킬 사용 이벤트 기록

시스템은 슬래시 커맨드로 시작하는 프롬프트를 감지하여 별도의 `skill_used` 이벤트를 기록(SHALL)해야 한다.

### Scenario: 슬래시 커맨드 감지

- **GIVEN** 사용자가 `/ts-init my-project` 프롬프트를 입력한 상태
- **WHEN** 훅이 실행되면
- **THEN** 기본 `prompt` 이벤트와 함께 `insertEvent({ v: 1, type: 'skill_used', ts, session_id, project, data: { skillName: 'ts-init' } })` 호출로 추가 기록(SHALL)된다

### Scenario: 일반 프롬프트 (비슬래시)

- **GIVEN** 사용자가 `"버그 수정해줘"` 프롬프트를 입력한 상태
- **WHEN** 훅이 실행되면
- **THEN** `skill_used` 이벤트는 기록하지 않고(SHALL) `prompt` 이벤트만 기록한다

---

## Requirement: REQ-DC-104 — Non-blocking 실행 보장

시스템은 어떤 오류가 발생해도 항상 exit code 0으로 종료(SHALL)해야 한다. 훅 실패가 Claude Code 세션을 방해해서는 안 된다.

### Scenario: 파일 시스템 오류 발생

- **GIVEN** 디스크 공간 부족 또는 권한 오류로 DB 쓰기가 실패하는 상태
- **WHEN** 훅 실행 중 예외가 발생하면
- **THEN** try-catch로 예외를 포착하고 `process.exit(0)`으로 정상 종료(SHALL)한다

### Scenario: 잘못된 stdin 입력

- **GIVEN** stdin에 잘못된 JSON이 전달된 상태
- **WHEN** `readStdin()` 파싱이 실패하면
- **THEN** 예외를 포착하고 `process.exit(0)`으로 정상 종료(SHALL)한다

---

## Requirement: REQ-DC-105 — 스킬 자동 감지 및 안내

시스템은 Phase 5 확장 시 사용자 프롬프트가 기존 스킬과 매칭되면 additionalContext로 스킬 사용을 안내해야 한다(SHALL).

### Scenario: 키워드 매칭 시 스킬 안내

- **GIVEN** `.claude/commands/ts-init.md` 스킬이 존재하고, 사용자가 "타입스크립트 프로젝트 초기화해줘" 프롬프트를 입력하면
- **WHEN** prompt-logger.mjs가 `matchSkill()`을 호출하여 50% 이상 매칭을 감지하면
- **THEN** stdout으로 `{ hookSpecificOutput: { hookEventName: 'UserPromptSubmit', additionalContext: '기존 스킬 /ts-init 이 있습니다...' } }`를 출력해야 한다

### Scenario: 매칭 없을 시 무출력

- **GIVEN** 프롬프트가 어떤 스킬과도 매칭되지 않으면
- **WHEN** matchSkill()이 null을 반환하면
- **THEN** additionalContext 출력 없이 정상 종료해야 한다

---

## 비고

- Phase 5 확장 시 `skill-matcher.mjs`를 import하여 커스텀 스킬 자동 감지 기능이 추가될 수 있음 (본 스펙 범위 외)
- 프롬프트 원문 수집 비활성화는 GDPR 등 규정 준수를 위한 선택적 기능
- `charCount`는 프라이버시 모드에서도 유지되어 프롬프트 길이 패턴 분석에 활용 가능
- 저장소가 JSONL에서 SQLite `events` 테이블로 변경됨. `db.mjs`의 `insertEvent()`를 사용하여 이벤트를 기록한다
