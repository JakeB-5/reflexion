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

> AI 분석 결과로 생성된 제안을 적용(apply)하거나 거부(dismiss)하는 CLI 도구. `bin/apply.mjs`는 3가지 제안 유형(skill, claude_md, hook)을 처리하고, `bin/dismiss.mjs`는 거부를 기록한다. 적용 시 `events` 테이블에 `skill_created` 또는 `suggestion_applied` 이벤트를 기록한다.

---

## 개요

사용자가 AI 분석으로 생성된 제안을 검토한 후, CLI를 통해 적용 또는 거부할 수 있는 인터페이스를 제공한다. 적용 시 제안 유형에 따라 커스텀 스킬 파일, CLAUDE.md 규칙, 훅 워크플로우 스크립트를 자동 생성한다. 모든 적용/거부 행위는 `feedback-tracker`의 `recordFeedback()`을 통해 기록되며, 적용 성공 시 `insertEvent()`로 이벤트를 추적한다.

### 대상 파일

| 파일 | 역할 |
|------|------|
| `bin/apply.mjs` | 제안 적용 도구 (skill/claude_md/hook 3가지 유형) |
| `bin/dismiss.mjs` | 제안 거부 기록 도구 |

### 의존 모듈

| import | 출처 |
|--------|------|
| `getCachedAnalysis` | `../lib/ai-analyzer.mjs` |
| `recordFeedback` | `../lib/feedback-tracker.mjs` |
| `insertEvent`, `getProjectName` | `../lib/db.mjs` |

---

## 요구사항

### REQ-SE-001: CLI 인자 파싱 및 제안 조회

시스템은 다음 CLI 인자를 파싱해야 한다(SHALL):
- `<suggestion-number>` — 적용할 제안 번호 (필수, 정수)
- `--global` — 전역 범위 적용 플래그
- `--project <name>` — 프로젝트 필터 (미지정 시 `basename(process.cwd())`로 추론)
- `--apply` — 훅 유형의 settings.json 자동 등록 플래그

시스템은 `getCachedAnalysis(168, project)` 함수를 호출하여 프로젝트 범위의 캐시된 분석 결과에서 제안 목록을 조회해야 한다(SHALL). 캐시가 없거나 제안이 비어 있으면 오류를 출력해야 한다(SHALL).

#### Scenario: 유효한 제안 선택

- **GIVEN** `getCachedAnalysis(168, project)`가 3개의 제안을 포함한 분석 결과를 반환할 때
- **WHEN** 사용자가 `node ~/.self-generation/bin/apply.mjs 2`를 실행하면
- **THEN** 2번째 제안이 선택되어 해당 유형의 적용 로직이 실행된다

#### Scenario: 프로젝트 필터 지정

- **GIVEN** `getCachedAnalysis(168, 'my-app')`가 유효한 분석 결과를 반환할 때
- **WHEN** 사용자가 `node ~/.self-generation/bin/apply.mjs 1 --project my-app`을 실행하면
- **THEN** `my-app` 프로젝트의 캐시된 분석 결과에서 1번째 제안이 선택된다

#### Scenario: 캐시 없음

- **GIVEN** `getCachedAnalysis(168, project)`가 `null`을 반환하거나 `suggestions`가 비어 있을 때
- **WHEN** 사용자가 `node ~/.self-generation/bin/apply.mjs 1`을 실행하면
- **THEN** "분석 결과가 없습니다. 먼저 node ~/.self-generation/bin/analyze.mjs 를 실행하세요." 오류 메시지를 출력하고 exit code 1로 종료한다

#### Scenario: 범위 초과

- **GIVEN** `getCachedAnalysis`가 3개의 제안을 반환할 때
- **WHEN** 사용자가 `node ~/.self-generation/bin/apply.mjs 5`를 실행하면
- **THEN** "유효한 범위: 1-3" 오류 메시지를 출력하고 exit code 1로 종료한다

#### Scenario: 번호 미지정 또는 비정수

- **GIVEN** 인자가 없거나 정수가 아닌 값이 전달될 때
- **WHEN** 사용자가 `node ~/.self-generation/bin/apply.mjs`를 실행하면
- **THEN** "사용법: node ~/.self-generation/bin/apply.mjs <번호> [--global]" 메시지를 출력하고 exit code 1로 종료한다

---

### REQ-SE-002: 스킬 제안 적용

`type: 'skill'` 제안의 경우, 시스템은 `.claude/commands/<skillName>.md` 파일을 생성해야 한다(SHALL). `skillName`이 없으면 `'auto-skill'`을 기본값으로 사용해야 한다(SHALL). `--global` 플래그 지정 시 `~/.claude/commands/` 경로에 생성해야 한다(SHALL). 디렉터리가 없으면 재귀적으로 생성해야 한다(SHALL).

생성되는 스킬 파일 내용은 다음 구조를 따라야 한다(SHALL):
- `# /<skillName>` 제목
- AI 감지 패턴 설명
- `## 감지된 패턴` 섹션 (`suggestion.evidence` 포함)
- `## 실행 지침` 섹션 (`suggestion.action` 또는 `$ARGUMENTS`)

#### Scenario: 프로젝트 스킬 생성

- **GIVEN** `type: 'skill'`, `skillName: 'ts-init'`, `action: 'TypeScript 프로젝트를 초기화합니다'`, `evidence: '3회 반복 감지'`인 제안이 선택되었을 때
- **WHEN** `--global` 플래그 없이 적용을 실행하면
- **THEN** `<cwd>/.claude/commands/ts-init.md` 파일이 생성되고, 감지된 패턴과 실행 지침이 포함된다

#### Scenario: 전역 스킬 생성

- **GIVEN** `type: 'skill'` 제안이 선택되었을 때
- **WHEN** `--global` 플래그와 함께 적용을 실행하면
- **THEN** `~/.claude/commands/<skillName>.md` 파일이 생성된다

#### Scenario: skillName 미지정

- **GIVEN** `type: 'skill'`이지만 `skillName` 필드가 없는 제안이 선택되었을 때
- **WHEN** 적용을 실행하면
- **THEN** `<baseDir>/auto-skill.md` 파일이 생성된다

---

### REQ-SE-003: CLAUDE.md 규칙 제안 적용

`type: 'claude_md'` 제안의 경우, 시스템은 대상 CLAUDE.md 파일에 규칙을 추가해야 한다(SHALL). 규칙 텍스트는 `suggestion.rule` 또는 `suggestion.summary`에서 가져와야 한다(SHALL). 추가 전 동일 규칙이 이미 존재하는지 중복 검사를 수행해야 한다(SHALL). `## 자동 감지된 규칙` 섹션이 없으면 생성해야 한다(SHALL).

`--global` 플래그에 따라 대상 파일을 결정해야 한다(SHALL):
- `--global`: `~/.claude/CLAUDE.md`
- 미지정: `<cwd>/.claude/CLAUDE.md`

대상 디렉터리가 없으면 재귀적으로 생성해야 한다(SHALL).

#### Scenario: 새 규칙 추가

- **GIVEN** `type: 'claude_md'`, `rule: 'TypeScript strict 모드를 항상 사용할 것'`인 제안이 선택되었고, CLAUDE.md에 해당 규칙이 없을 때
- **WHEN** 적용을 실행하면
- **THEN** CLAUDE.md의 `## 자동 감지된 규칙` 섹션에 `- TypeScript strict 모드를 항상 사용할 것`이 추가된다

#### Scenario: 중복 규칙 방지

- **GIVEN** `type: 'claude_md'` 제안이 선택되었고, CLAUDE.md에 이미 동일한 규칙 텍스트가 존재할 때
- **WHEN** 적용을 실행하면
- **THEN** "이미 동일한 규칙이 존재합니다." 메시지를 출력하고 파일을 수정하지 않는다

#### Scenario: CLAUDE.md 파일 미존재

- **GIVEN** 대상 경로에 CLAUDE.md 파일이 없을 때
- **WHEN** 적용을 실행하면
- **THEN** 디렉터리를 생성하고, `## 자동 감지된 규칙` 섹션과 함께 새 파일을 생성한다

---

### REQ-SE-004: 훅 워크플로우 제안 적용

`type: 'hook'` 제안의 경우, 시스템은 `~/.self-generation/hooks/auto/workflow-<id>.mjs` 파일을 생성해야 한다(SHALL). 디렉터리가 없으면 재귀적으로 생성해야 한다(SHALL).

`suggestion.hookCode`가 존재하면 해당 코드를 파일로 저장해야 한다(SHALL). `--apply` 플래그 지정 시 `~/.claude/settings.json`에 훅을 자동 등록해야 한다(SHALL). `--apply` 없이는 등록 안내와 자동 등록 명령어를 출력해야 한다(SHOULD).

훅 이벤트는 `suggestion.hookEvent`를 사용하며, 미지정 시 `'PostToolUse'`를 기본값으로 사용해야 한다(SHALL).

#### Scenario: 훅 스크립트 생성 (수동 등록)

- **GIVEN** `type: 'hook'`, `hookCode`가 포함된 제안이 선택되었을 때
- **WHEN** `--apply` 플래그 없이 적용을 실행하면
- **THEN** `hooks/auto/workflow-<id>.mjs` 파일이 생성되고, `settings.json` 등록 안내와 자동 등록 명령어(`node ~/.self-generation/bin/apply.mjs <id> --apply`)가 출력된다

#### Scenario: 훅 자동 등록

- **GIVEN** `type: 'hook'`, `hookCode`와 `hookEvent: 'PostToolUse'`가 포함된 제안이 선택되었을 때
- **WHEN** `--apply` 플래그와 함께 적용을 실행하면
- **THEN** `hooks/auto/workflow-<id>.mjs` 파일이 생성되고, `~/.claude/settings.json`의 `PostToolUse` 이벤트에 훅이 등록되며, 기존 settings.json 구조가 보존된다

#### Scenario: hookCode 미포함

- **GIVEN** `type: 'hook'`이지만 `hookCode` 필드가 없는 제안이 선택되었을 때
- **WHEN** 적용을 실행하면
- **THEN** "훅 코드 미생성 — 프롬프트 템플릿에 hookCode 필드를 요청하세요" 경고 메시지를 출력한다

---

### REQ-SE-005: 제안 거부 기록

`bin/dismiss.mjs`는 사용자가 지정한 제안 ID에 대해 `recordFeedback(id, 'rejected', { suggestionType: 'unknown' })`를 호출하여 거부를 기록해야 한다(SHALL). 거부 확인 메시지와 향후 AI 분석 제외 안내를 출력해야 한다(SHALL).

#### Scenario: 정상 거부

- **GIVEN** 유효한 제안 ID `suggestion-abc-123`이 존재할 때
- **WHEN** `node ~/.self-generation/bin/dismiss.mjs suggestion-abc-123`을 실행하면
- **THEN** `feedback` 테이블에 `{ action: 'rejected', suggestion_id: 'suggestion-abc-123', suggestion_type: 'unknown' }` 레코드가 INSERT되고, "제안 거부 기록됨: suggestion-abc-123" 확인 메시지와 "이 패턴은 향후 AI 분석 시 제외 컨텍스트로 전달됩니다." 안내가 출력된다

#### Scenario: ID 미지정

- **GIVEN** 인자 없이 실행할 때
- **WHEN** `node ~/.self-generation/bin/dismiss.mjs`를 실행하면
- **THEN** "사용법: node ~/.self-generation/bin/dismiss.mjs <suggestion-id>" 안내를 출력하고 exit code 1로 종료한다

---

### REQ-SE-006: 피드백 기록 연동

모든 적용(apply) 성공 시 `recordFeedback(id, 'accepted', { suggestionType, summary })`를 호출해야 한다(SHALL). 피드백 기록은 제안 유형별 적용 로직(switch 분기) 이후에 공통으로 실행해야 한다(SHALL).

#### Scenario: 적용 성공 시 피드백 기록

- **GIVEN** `type: 'skill'` 제안이 성공적으로 적용되었을 때
- **WHEN** 스킬 파일 생성이 완료되면
- **THEN** `recordFeedback(suggestion.id, 'accepted', { suggestionType: 'skill', summary: suggestion.summary })`가 호출되어 `feedback` 테이블에 INSERT된다

---

### REQ-SE-007: 이벤트 추적 (skill_created / suggestion_applied)

모든 적용(apply) 성공 시 `insertEvent()`를 호출하여 `events` 테이블에 이벤트를 기록해야 한다(SHALL). 제안 유형이 `'skill'`이면 `type: 'skill_created'`, 그 외에는 `type: 'suggestion_applied'`로 기록해야 한다(SHALL).

이벤트 데이터에는 다음 필드를 포함해야 한다(SHALL):
- `v: 1` — 스키마 버전
- `type` — 이벤트 유형 (`skill_created` 또는 `suggestion_applied`)
- `ts` — ISO 8601 타임스탬프
- `project` — `getProjectName(process.cwd())`로 결정
- `data` — `{ suggestionId, suggestionType, scope }` (scope: `'global'` 또는 `'project'`)

#### Scenario: 스킬 적용 시 skill_created 이벤트

- **GIVEN** `type: 'skill'` 제안이 성공적으로 적용되었을 때
- **WHEN** 피드백 기록 후 이벤트 추적이 실행되면
- **THEN** `events` 테이블에 `{ v: 1, type: 'skill_created', project: '<project>', data: { suggestionId: '<id>', suggestionType: 'skill', scope: 'project' } }` 이벤트가 INSERT된다

#### Scenario: CLAUDE.md 적용 시 suggestion_applied 이벤트

- **GIVEN** `type: 'claude_md'` 제안이 성공적으로 적용되었을 때
- **WHEN** 피드백 기록 후 이벤트 추적이 실행되면
- **THEN** `events` 테이블에 `{ v: 1, type: 'suggestion_applied', project: '<project>', data: { suggestionId: '<id>', suggestionType: 'claude_md', scope: 'project' } }` 이벤트가 INSERT된다

#### Scenario: 전역 적용 시 scope 반영

- **GIVEN** 제안이 `--global` 플래그와 함께 적용되었을 때
- **WHEN** 이벤트 추적이 실행되면
- **THEN** `data.scope`가 `'global'`로 기록된다

---

## 비기능 요구사항

### 성능

- CLI 실행 시간: 500ms 이내 (SHOULD)
- 파일 I/O는 동기 API 사용 (Node.js built-in `fs`, `path` 모듈)

### 보안

- 훅 자동 등록(`--apply`)은 기존 `settings.json` 구조를 보존해야 한다(SHALL)
- 사용자 승인 없이 자동 적용이 발생해서는 안 된다(SHALL NOT)

---

## 제약사항

- `better-sqlite3` 외 추가 npm 패키지 사용 금지
- ES Modules 형식 (`.mjs` 확장자)
- `analysis_cache` 테이블의 제안 스키마는 `ai-analyzer` 모듈이 정의함
- 캐시 조회는 `getCachedAnalysis(maxAgeHours, project)` 함수를 통해 수행 (직접 SQL 접근 금지)
- `recordFeedback()` 함수는 `feedback-tracker` 모듈에서 import
- `insertEvent()`, `getProjectName()` 함수는 `db` 모듈에서 import
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
| skill_created | 스킬 제안 적용 시 `events` 테이블에 기록되는 이벤트 유형 |
| suggestion_applied | 비스킬 제안 적용 시 `events` 테이블에 기록되는 이벤트 유형 |
