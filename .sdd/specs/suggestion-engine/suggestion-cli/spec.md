---
id: suggestion-cli
title: "제안 적용/거부 CLI 도구"
status: draft
created: 2026-02-07
domain: suggestion-engine
depends: "ai-analysis/ai-analyzer, suggestion-engine/feedback-tracker, data-collection/log-writer"
constitution_version: "2.0.0"
---

# suggestion-cli

> AI 분석 결과로 생성된 제안을 적용(apply)하거나 거부(dismiss)하는 CLI 도구. `bin/apply.mjs`는 3가지 제안 유형(skill, claude_md, hook)을 처리하고, `bin/dismiss.mjs`는 거부를 기록한다.

---

## 개요

사용자가 AI 분석으로 생성된 제안을 검토한 후, CLI를 통해 적용 또는 거부할 수 있는 인터페이스를 제공한다. 적용 시 제안 유형에 따라 커스텀 스킬 파일, CLAUDE.md 규칙, 훅 워크플로우 스크립트를 자동 생성한다. 모든 적용/거부 행위는 `feedback-tracker`의 `recordFeedback()`을 통해 기록된다.

### 대상 파일

| 파일 | 역할 |
|------|------|
| `bin/apply.mjs` | 제안 적용 도구 (skill/claude_md/hook 3가지 유형) |
| `bin/dismiss.mjs` | 제안 거부 기록 도구 |

---

## 요구사항

### REQ-SE-001: 제안 조회 및 유효성 검증

시스템은 `analysis_cache` 테이블에서 제안 목록을 조회하고, 사용자가 지정한 번호/ID로 특정 제안을 선택할 수 있어야 한다(SHALL). 캐시가 없거나 유효 기간(7일)이 초과된 경우 오류를 출력해야 한다(SHALL).

```sql
SELECT analysis FROM analysis_cache
WHERE ts > datetime('now', '-7 days')
ORDER BY ts DESC LIMIT 1
```

#### Scenario: 유효한 제안 선택

- **GIVEN** `analysis_cache` 테이블에 3개의 제안이 캐시되어 있고, 캐시 생성 시점이 7일 이내일 때
- **WHEN** 사용자가 `node ~/.self-generation/bin/apply.mjs 2`를 실행하면
- **THEN** 2번째 제안이 선택되어 해당 유형의 적용 로직이 실행된다

#### Scenario: 캐시 없음

- **GIVEN** `analysis_cache` 테이블이 비어 있거나 DB가 존재하지 않을 때
- **WHEN** 사용자가 `node ~/.self-generation/bin/apply.mjs 1`을 실행하면
- **THEN** "분석 결과가 없습니다" 오류 메시지를 출력하고 exit code 1로 종료한다

#### Scenario: 범위 초과

- **GIVEN** `analysis_cache` 테이블에 3개의 제안이 있을 때
- **WHEN** 사용자가 `node ~/.self-generation/bin/apply.mjs 5`를 실행하면
- **THEN** "유효한 범위: 1-3" 오류 메시지를 출력하고 exit code 1로 종료한다

---

### REQ-SE-002: 스킬 제안 적용

`type: 'skill'` 제안의 경우, 시스템은 `.claude/commands/<skillName>.md` 파일을 생성해야 한다(SHALL). `--global` 플래그 지정 시 `~/.claude/commands/` 경로에 생성해야 한다(SHALL). 디렉터리가 없으면 재귀적으로 생성해야 한다(SHALL).

#### Scenario: 프로젝트 스킬 생성

- **GIVEN** `type: 'skill'`, `skillName: 'ts-init'`, `action: 'TypeScript 프로젝트를 초기화합니다'`인 제안이 선택되었을 때
- **WHEN** `--global` 플래그 없이 적용을 실행하면
- **THEN** `<cwd>/.claude/commands/ts-init.md` 파일이 생성되고, 감지된 패턴과 실행 지침이 포함된다

#### Scenario: 전역 스킬 생성

- **GIVEN** `type: 'skill'` 제안이 선택되었을 때
- **WHEN** `--global` 플래그와 함께 적용을 실행하면
- **THEN** `~/.claude/commands/<skillName>.md` 파일이 생성된다

---

### REQ-SE-003: CLAUDE.md 규칙 제안 적용

`type: 'claude_md'` 제안의 경우, 시스템은 대상 CLAUDE.md 파일에 규칙을 추가해야 한다(SHALL). 추가 전 동일 규칙이 이미 존재하는지 중복 검사를 수행해야 한다(SHALL). `## 자동 감지된 규칙` 섹션이 없으면 생성해야 한다(SHALL).

#### Scenario: 새 규칙 추가

- **GIVEN** `type: 'claude_md'`, `rule: 'TypeScript strict 모드를 항상 사용할 것'`인 제안이 선택되었고, CLAUDE.md에 해당 규칙이 없을 때
- **WHEN** 적용을 실행하면
- **THEN** CLAUDE.md의 `## 자동 감지된 규칙` 섹션에 `- TypeScript strict 모드를 항상 사용할 것`이 추가된다

#### Scenario: 중복 규칙 방지

- **GIVEN** `type: 'claude_md'` 제안이 선택되었고, CLAUDE.md에 이미 동일한 규칙 텍스트가 존재할 때
- **WHEN** 적용을 실행하면
- **THEN** "이미 동일한 규칙이 존재합니다" 메시지를 출력하고 파일을 수정하지 않는다

---

### REQ-SE-004: 훅 워크플로우 제안 적용

`type: 'hook'` 제안의 경우, 시스템은 `hooks/auto/workflow-<id>.mjs` 파일을 생성해야 한다(SHALL). `--apply` 플래그 지정 시 `~/.claude/settings.json`에 훅을 자동 등록해야 한다(SHALL). `--apply` 없이는 등록 안내만 출력해야 한다(SHOULD).

#### Scenario: 훅 스크립트 생성 (수동 등록)

- **GIVEN** `type: 'hook'`, `hookCode`가 포함된 제안이 선택되었을 때
- **WHEN** `--apply` 플래그 없이 적용을 실행하면
- **THEN** `hooks/auto/workflow-<id>.mjs` 파일이 생성되고, `settings.json` 등록 안내가 출력된다

#### Scenario: 훅 자동 등록

- **GIVEN** `type: 'hook'`, `hookCode`와 `hookEvent: 'PostToolUse'`가 포함된 제안이 선택되었을 때
- **WHEN** `--apply` 플래그와 함께 적용을 실행하면
- **THEN** `hooks/auto/workflow-<id>.mjs` 파일이 생성되고, `~/.claude/settings.json`의 `PostToolUse` 이벤트에 훅이 등록된다

#### Scenario: hookCode 미포함

- **GIVEN** `type: 'hook'`이지만 `hookCode` 필드가 없는 제안이 선택되었을 때
- **WHEN** 적용을 실행하면
- **THEN** "훅 코드 미생성" 경고 메시지를 출력한다

---

### REQ-SE-005: 제안 거부 기록

`bin/dismiss.mjs`는 사용자가 지정한 제안 ID에 대해 `recordFeedback(id, 'rejected')`를 호출하여 거부를 기록해야 한다(SHALL). 거부된 제안은 향후 AI 분석 시 제외 컨텍스트로 전달된다.

#### Scenario: 정상 거부

- **GIVEN** 유효한 제안 ID `suggestion-abc-123`이 존재할 때
- **WHEN** `node ~/.self-generation/bin/dismiss.mjs suggestion-abc-123`을 실행하면
- **THEN** `feedback` 테이블에 `{ action: 'rejected', suggestion_id: 'suggestion-abc-123' }` 레코드가 INSERT되고, 확인 메시지가 출력된다

#### Scenario: ID 미지정

- **GIVEN** 인자 없이 실행할 때
- **WHEN** `node ~/.self-generation/bin/dismiss.mjs`를 실행하면
- **THEN** 사용법 안내를 출력하고 exit code 1로 종료한다

---

### REQ-SE-006: 피드백 기록 연동

모든 적용(apply) 성공 시 `recordFeedback(id, 'accepted', { suggestionType, summary })`를 호출해야 한다(SHALL). 모든 거부(dismiss) 시 `recordFeedback(id, 'rejected', { suggestionType: 'unknown' })`를 호출해야 한다(SHALL).

#### Scenario: 적용 성공 시 피드백 기록

- **GIVEN** `type: 'skill'` 제안이 성공적으로 적용되었을 때
- **WHEN** 스킬 파일 생성이 완료되면
- **THEN** `recordFeedback(suggestion.id, 'accepted', { suggestionType: 'skill', summary: suggestion.summary })`가 호출되어 `feedback` 테이블에 INSERT된다

---

## 비기능 요구사항

### 성능

- CLI 실행 시간: 500ms 이내 (SHOULD)
- 파일 I/O는 동기 API 사용 (Node.js built-in만 사용)

### 보안

- 훅 자동 등록(`--apply`)은 기존 `settings.json` 구조를 보존해야 한다(SHALL)
- 사용자 승인 없이 자동 적용이 발생해서는 안 된다(SHALL NOT)

---

## 제약사항

- `better-sqlite3` 외 추가 npm 패키지 사용 금지
- ES Modules 형식 (`.mjs` 확장자)
- `analysis_cache` 테이블의 제안 스키마는 `ai-analyzer` 모듈이 정의함
- `recordFeedback()` 함수는 `feedback-tracker` 모듈에서 import (내부적으로 `feedback` 테이블에 INSERT)
- 데이터 접근은 `lib/db.mjs`를 통해 SQLite DB(`self-gen.db`)에서 수행한다

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| 제안(suggestion) | AI 분석이 생성한 개선 항목 (skill, claude_md, hook 3가지 유형) |
| 적용(apply) | 제안을 실제 파일로 생성/수정하는 행위 |
| 거부(dismiss) | 제안을 채택하지 않고 기록만 남기는 행위 |
| 전역 범위(global) | `~/.claude/` 하위에 적용 (모든 프로젝트에 영향) |
| 프로젝트 범위(project) | `<cwd>/.claude/` 하위에 적용 (현재 프로젝트에만 영향) |
