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

> UserPromptSubmit 훅 스크립트 (`hooks/prompt-logger.mjs`). 사용자 프롬프트를 수집하여 `events` 테이블에 기록하고, 프라이버시 태그를 스트리핑하며, 기존 스킬과의 벡터/키워드 매칭을 수행하고, 슬래시 커맨드(`/`) 사용 시 스킬 사용 이벤트를 별도 기록한다. v6 확장 버전(DESIGN.md 8.2절)이 최종 구현이다.

---

## Requirement: REQ-DC-101 — 프롬프트 수집

시스템은 UserPromptSubmit 이벤트 발생 시 프롬프트 데이터를 `events` 테이블에 기록(SHALL)해야 한다. 공통 필드는 각 컬럼에, 프롬프트 고유 필드(`text`, `charCount`)는 `data` JSON 컬럼에 저장된다.

### Scenario: 일반 프롬프트 기록

- **GIVEN** 사용자가 Claude Code에 프롬프트를 입력한 상태
- **WHEN** UserPromptSubmit 훅이 트리거되면
- **THEN** 다음 필드를 포함하는 이벤트가 `insertEvent()`로 기록(SHALL)된다: `v: 1`, `type: 'prompt'`, `ts` (ISO 8601), `sessionId`, `project` (디렉토리명), `projectPath` (전체 경로), `text` (프라이버시 처리 완료), `charCount`

### Scenario: stdin 필드 매핑

- **GIVEN** Claude Code가 stdin으로 `{ prompt, session_id, cwd }`를 전달하는 상태
- **WHEN** 훅이 실행되면
- **THEN** `input.session_id` → `sessionId`, `getProjectPath(input.cwd)` → `projectPath`, `getProjectName(getProjectPath(input.cwd))` → `project`로 매핑(SHALL)된다

---

## Requirement: REQ-DC-102 — 프라이버시 보호 (프롬프트 텍스트 수집 제어)

시스템은 설정에 따라 프롬프트 원문 수집을 비활성화(SHALL)할 수 있어야 하며, `<private>` 태그 스트리핑을 적용해야 한다. 처리 순서: `collectPromptText` 확인 → `stripPrivateTags()` 적용.

### Scenario: collectPromptText=false 설정 시

- **GIVEN** `config.json`에서 `collectPromptText: false`로 설정된 상태
- **WHEN** 프롬프트가 기록되면
- **THEN** `rawPrompt`가 `'[REDACTED]'`로 대체(SHALL)된 후 `stripPrivateTags()`가 적용되고, `charCount`는 처리된 텍스트의 길이를 사용한다

### Scenario: 기본값 (수집 활성화) + private 태그 스트리핑

- **GIVEN** `config.json`이 없거나 `collectPromptText`가 명시되지 않은 상태
- **WHEN** 프롬프트가 기록되면
- **THEN** `input.prompt`에 `stripPrivateTags()`를 적용하여 `<private>...</private>` 태그 내용을 `[PRIVATE]`로 치환(SHALL)한 후 `text` 필드에 저장된다

---

## Requirement: REQ-DC-103 — 스킬 사용 이벤트 기록

시스템은 슬래시 커맨드로 시작하는 프롬프트를 감지하여 실제 스킬 이름과 일치하는 경우에만 별도의 `skill_used` 이벤트를 기록(SHALL)해야 한다.

### Scenario: 실제 스킬 사용 감지 (v7 P5)

- **GIVEN** 사용자가 `/ts-init my-project` 프롬프트를 입력하고, `loadSkills()`가 반환한 스킬 목록에 `ts-init`이 존재하는 상태
- **WHEN** 훅이 실행되면
- **THEN** 기본 `prompt` 이벤트와 함께 `insertEvent({ v: 1, type: 'skill_used', ts, sessionId, project, projectPath, skillName: 'ts-init' })` 호출로 추가 기록(SHALL)된다

### Scenario: 슬래시 접두사이지만 실제 스킬이 아닌 경우

- **GIVEN** 사용자가 `/usr/bin/node` 등 경로를 입력한 상태
- **WHEN** 훅이 실행되면
- **THEN** `loadSkills()`의 스킬 목록에 없으므로 `skill_used` 이벤트를 기록하지 않는다(SHALL)

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

## Requirement: REQ-DC-105 — 시스템 활성화 체크

시스템은 훅 실행 초기에 `isEnabled()` (또는 `loadConfig().enabled`)를 확인하여 비활성화 시 즉시 종료(SHALL)해야 한다.

### Scenario: 시스템 비활성화

- **GIVEN** `config.json`에서 `enabled: false`로 설정된 상태
- **WHEN** 훅이 실행되면
- **THEN** `config.enabled === false` 확인 후 즉시 `process.exit(0)`으로 종료(SHALL)한다

---

## Requirement: REQ-DC-106 — 스킬 자동 감지 및 안내

시스템은 사용자 프롬프트가 기존 스킬과 매칭되면 `additionalContext`로 스킬 사용을 안내(SHALL)해야 한다. 벡터 유사도 검색을 우선 시도하고, 실패 시 키워드 매칭으로 폴백한다. 데몬 콜드 스타트 방지를 위해 2초 타임아웃을 적용한다.

### Scenario: 벡터 매칭 시 스킬 안내

- **GIVEN** `skill_embeddings` 테이블에 스킬 임베딩이 존재하고, 프롬프트와 distance < 0.76인 스킬이 있는 상태
- **WHEN** `matchSkill(input.prompt, skills)`가 벡터 매칭 결과를 반환하면
- **THEN** stdout으로 `{ hookSpecificOutput: { hookEventName: 'UserPromptSubmit', additionalContext: '[Reflexion] 이 작업과 관련된 커스텀 스킬이 있습니다: /<name> (전역/프로젝트 스킬)\n사용자에게 이 스킬 사용을 제안해주세요.' } }`를 출력(SHALL)한다

### Scenario: 키워드 폴백 매칭

- **GIVEN** 벡터 검색이 실패하거나 임베딩이 없는 상태
- **WHEN** `matchSkill()`이 키워드 폴백으로 50% 이상 매칭을 감지하면
- **THEN** 동일한 `additionalContext` 형식으로 스킬을 안내(SHALL)한다

### Scenario: 2초 타임아웃 적용 (v9)

- **GIVEN** 임베딩 데몬이 콜드 스타트 중인 상태
- **WHEN** `matchSkill()` 호출이 2초를 초과하면
- **THEN** `Promise.race()`로 타임아웃하여 `null`을 반환하고 스킬 안내 없이 종료(SHALL)한다

### Scenario: 매칭 없을 시 무출력

- **GIVEN** 프롬프트가 어떤 스킬과도 매칭되지 않으면
- **WHEN** `matchSkill()`이 `null`을 반환하면
- **THEN** `additionalContext` 출력 없이 정상 종료(SHALL)한다

### Scenario: 스킬 미존재 시 매칭 생략

- **GIVEN** `loadSkills(input.cwd)`가 빈 배열을 반환하는 상태
- **WHEN** 훅이 실행되면
- **THEN** `matchSkill()` 호출을 건너뛰고 정상 종료(SHALL)한다

---

## 비고

- **v6 확장 버전이 최종본**: DESIGN.md 4.3절(Phase 1 기본)이 아닌 8.2절(v6 확장)을 구현한다. Phase 1과 v6를 병합하지 말 것
- `loadSkills()`는 `skill-matcher.mjs`에서 import하며, 전역(`~/.claude/commands/`) + 프로젝트(`<cwd>/.claude/commands/`) 스킬을 로드한다
- `matchSkill()`은 벡터 유사도 검색(distance < 0.76)을 우선 시도하고, 실패 시 키워드 50% 매칭으로 폴백한다
- `charCount`는 프라이버시 처리 후 텍스트의 길이를 사용한다
- 저장소가 JSONL에서 SQLite `events` 테이블로 변경됨. `db.mjs`의 `insertEvent()`를 사용하여 이벤트를 기록한다
- 프롬프트 원문 수집 비활성화는 GDPR 등 규정 준수를 위한 선택적 기능
