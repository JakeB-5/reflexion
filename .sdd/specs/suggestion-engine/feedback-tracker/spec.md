---
id: feedback-tracker
title: "피드백 추적 및 효과성 메트릭"
status: draft
created: 2026-02-07
domain: suggestion-engine
depends: "data-collection/log-writer, realtime-assist/skill-matcher"
constitution_version: "2.0.0"
---

# feedback-tracker

> 제안 채택/거부 피드백을 `feedback` 테이블에 기록하고, JavaScript 집계로 요약을 생성하며, `events` 테이블 기반으로 스킬 사용률/규칙 효과성/미사용 스킬 등 효과성 메트릭을 산출하는 유틸리티 모듈. AI 분석 프롬프트에 피드백 컨텍스트를 주입하여 제안 품질을 자체 개선한다. DB 접근은 `lib/db.mjs`를 통한다.

---

## 개요

`lib/feedback-tracker.mjs`는 제안 적용/거부 이력을 `feedback` 테이블에 기록하고, JavaScript 기반 집계를 통해 AI 분석 시 주입할 피드백 요약을 생성한다. 정적 임계값 대신 피드백 데이터를 AI 프롬프트 컨텍스트로 전달하여 Claude가 사용자 선호도를 직접 학습하게 한다.

### 대상 파일/테이블

| 대상 | 역할 |
|------|------|
| `lib/feedback-tracker.mjs` | 피드백 기록, 요약, 효과성 메트릭 |
| `feedback` 테이블 (`self-gen.db`) | 피드백 데이터 저장소 |
| `events` 테이블 (`self-gen.db`) | 스킬 사용/에러 이벤트 조회 대상 |

### 의존 모듈

| import | 출처 |
|--------|------|
| `getDb`, `queryEvents` | `./db.mjs` |
| `loadSkills` | `./skill-matcher.mjs` |

### 함수 목록

| 함수 | 공개 여부 | 설명 |
|------|-----------|------|
| `recordFeedback(id, action, meta)` | `export` | 피드백 엔트리 기록 |
| `getFeedbackSummary()` | `export` | AI 프롬프트 주입용 요약 생성 |
| `calcSkillUsageRate()` | 내부 함수 | 스킬 실제 사용률 계산 |
| `calcRuleEffectiveness()` | 내부 함수 | 규칙 추가 후 에러 재발 여부 측정 |
| `findStaleSkills(days, projectPath)` | 내부 함수 | 장기 미사용 스킬 탐지 |

**비고**: `calcSkillUsageRate`, `calcRuleEffectiveness`, `findStaleSkills`는 export하지 않으며 `getFeedbackSummary()` 내부에서만 호출된다.

---

## 요구사항

### REQ-SE-101: 피드백 기록

시스템은 `recordFeedback(suggestionId, action, details)` 함수를 export하여 피드백을 `feedback` 테이블에 INSERT해야 한다(SHALL). `action`은 `'accepted'`, `'rejected'`, `'dismissed'` 중 하나여야 한다(SHALL).

```sql
INSERT INTO feedback (v, ts, suggestion_id, action, suggestion_type, summary)
VALUES (1, ?, ?, ?, ?, ?)
```

`details` 객체의 필드 매핑:
- `details.suggestionType` → `suggestion_type` (미지정 시 `null`)
- `details.summary` → `summary` (미지정 시 `null`)

#### Scenario: 채택 피드백 기록

- **GIVEN** 제안 ID `sg-001`이 스킬 유형으로 적용되었을 때
- **WHEN** `recordFeedback('sg-001', 'accepted', { suggestionType: 'skill', summary: 'TS 초기화 스킬' })`이 호출되면
- **THEN** `feedback` 테이블에 `{ v: 1, ts: '<ISO>', suggestion_id: 'sg-001', action: 'accepted', suggestion_type: 'skill', summary: 'TS 초기화 스킬' }` 레코드가 INSERT된다

#### Scenario: 거부 피드백 기록

- **GIVEN** 제안 ID `sg-002`가 거부되었을 때
- **WHEN** `recordFeedback('sg-002', 'rejected', { suggestionType: 'unknown' })`이 호출되면
- **THEN** `feedback` 테이블에 `{ v: 1, ts: '<ISO>', suggestion_id: 'sg-002', action: 'rejected', suggestion_type: 'unknown', summary: null }` 레코드가 INSERT된다

#### Scenario: details 미지정

- **GIVEN** details 인자가 생략될 때
- **WHEN** `recordFeedback('sg-003', 'dismissed')`이 호출되면
- **THEN** `feedback` 테이블에 `{ v: 1, ts: '<ISO>', suggestion_id: 'sg-003', action: 'dismissed', suggestion_type: null, summary: null }` 레코드가 INSERT된다

---

### REQ-SE-102: 피드백 요약 생성

시스템은 `getFeedbackSummary()` 함수를 export하여 AI 분석 프롬프트에 주입할 피드백 요약을 반환해야 한다(SHALL).

구현 방식: `SELECT * FROM feedback ORDER BY ts ASC`로 전체 레코드를 조회한 후 JavaScript로 필터링/집계해야 한다(SHALL). SQL 집계가 아닌 JavaScript 기반 처리를 사용한다.

반환 객체 필드:

| 필드 | 타입 | 설명 |
|------|------|------|
| `total` | `number` | 전체 피드백 수 |
| `acceptedCount` | `number` | `action === 'accepted'` 레코드 수 |
| `rejectedCount` | `number` | `action === 'rejected'` 또는 `'dismissed'` 레코드 수 |
| `rate` | `number` | `acceptedCount / total` (0~1) |
| `recentRejections` | `string[]` | 최근 거부 10건의 `summary` 또는 `suggestion_id` |
| `recentAcceptances` | `string[]` | 최근 채택 10건의 `summary` 또는 `suggestion_id` |
| `skillUsageRate` | `number\|null` | `calcSkillUsageRate()` 결과 |
| `ruleEffectiveness` | `object\|null` | `calcRuleEffectiveness()` 결과 |
| `staleSkills` | `string[]` | `findStaleSkills(30)` 결과 |

`recentRejections`와 `recentAcceptances`는 ASC 정렬된 배열의 마지막 10건(`.slice(-10)`)을 사용하고, 각 항목은 `e.summary || e.suggestion_id`로 매핑해야 한다(SHALL).

#### Scenario: 정상 요약 생성

- **GIVEN** `feedback` 테이블에 채택 7건, 거부 3건이 기록되어 있을 때
- **WHEN** `getFeedbackSummary()`를 호출하면
- **THEN** `{ total: 10, acceptedCount: 7, rejectedCount: 3, rate: 0.7, recentRejections: [...], recentAcceptances: [...], skillUsageRate: <number|null>, ruleEffectiveness: <object|null>, staleSkills: [...] }`를 반환한다

#### Scenario: 피드백 레코드 없음

- **GIVEN** `feedback` 테이블이 비어 있을 때
- **WHEN** `getFeedbackSummary()`를 호출하면
- **THEN** `null`을 반환한다

#### Scenario: 최근 이력 제한

- **GIVEN** `feedback` 테이블에 채택 15건, 거부 12건이 기록되어 있을 때
- **WHEN** `getFeedbackSummary()`를 호출하면
- **THEN** `recentRejections`는 가장 최근 10건만, `recentAcceptances`는 가장 최근 10건만 포함한다 (ASC 정렬 후 `.slice(-10)`)

#### Scenario: DB 접근 실패

- **GIVEN** `self-gen.db` 파일이 존재하지 않거나 접근할 수 없을 때
- **WHEN** `getFeedbackSummary()`를 호출하면
- **THEN** 예외를 throw하지 않고 `null`을 반환한다

---

### REQ-SE-103: 스킬 사용률 계산 (내부 함수)

시스템은 `calcSkillUsageRate()` 내부 함수를 통해 생성된 스킬의 실제 사용률을 계산해야 한다(SHALL). `events` 테이블에서 SQL COUNT로 `type = 'skill_used'` 이벤트 수를 `type = 'skill_created'` 이벤트 수로 나눈 비율을 반환해야 한다(SHALL).

```sql
SELECT COUNT(*) AS cnt FROM events WHERE type = 'skill_used'
SELECT COUNT(*) AS cnt FROM events WHERE type = 'skill_created'
```

`skill_created`가 0이면 `null`을 반환해야 한다(SHALL).

#### Scenario: 사용률 정상 계산

- **GIVEN** `events` 테이블에 `skill_created` 이벤트 4건, `skill_used` 이벤트 10건이 있을 때
- **WHEN** `calcSkillUsageRate()`를 호출하면
- **THEN** `2.5`를 반환한다 (10 / 4)

#### Scenario: 생성 이력 없음

- **GIVEN** `events` 테이블에 `skill_created` 이벤트가 없을 때
- **WHEN** `calcSkillUsageRate()`를 호출하면
- **THEN** `null`을 반환한다

#### Scenario: DB 접근 실패

- **GIVEN** `self-gen.db` 파일이 존재하지 않거나 접근할 수 없을 때
- **WHEN** `calcSkillUsageRate()`를 호출하면
- **THEN** `null`을 반환한다

---

### REQ-SE-104: 규칙 효과성 측정 (내부 함수)

시스템은 `calcRuleEffectiveness()` 내부 함수를 통해 규칙 추가 후 에러 재발 여부를 측정해야 한다(SHALL). `events` 테이블에서 `type = 'tool_error'` 이벤트의 전체 수와 최근 7일 내 발생 수를 반환해야 한다(SHALL).

최근 7일 기준은 JavaScript `Date.now() - 7 * 86400000`으로 계산하여 ISO 8601 문자열로 변환 후 SQL 파라미터로 전달해야 한다(SHALL).

```javascript
const recentCutoff = new Date(Date.now() - 7 * 86400000).toISOString();
```

```sql
SELECT COUNT(*) AS cnt FROM events WHERE type = 'tool_error'
SELECT COUNT(*) AS cnt FROM events WHERE type = 'tool_error' AND ts >= ?
```

#### Scenario: 효과성 정상 측정

- **GIVEN** `events` 테이블에 총 20건의 `tool_error`가 있고, 그 중 3건이 최근 7일 내 발생했을 때
- **WHEN** `calcRuleEffectiveness()`를 호출하면
- **THEN** `{ totalErrors: 20, recentErrors: 3 }`을 반환한다

#### Scenario: DB 접근 실패

- **GIVEN** `self-gen.db` 파일이 존재하지 않거나 접근할 수 없을 때
- **WHEN** `calcRuleEffectiveness()`를 호출하면
- **THEN** `null`을 반환한다

---

### REQ-SE-105: 미사용 스킬 탐지 (내부 함수)

시스템은 `findStaleSkills(days, projectPath = null)` 내부 함수를 통해 지정된 기간(기본 30일) 동안 사용되지 않은 스킬을 탐지해야 한다(SHALL).

`loadSkills(projectPath)`로 등록된 스킬 목록을 가져오고, 각 스킬에 대해 `events` 테이블에서 최근 사용 이력을 SQL로 조회하여 미사용 스킬 이름 배열을 반환해야 한다(SHALL).

기간 기준은 JavaScript `Date.now() - days * 86400000`으로 계산하여 ISO 8601 문자열로 변환해야 한다(SHALL).

```javascript
const threshold = new Date(Date.now() - days * 86400000).toISOString();
```

```sql
SELECT ts FROM events
WHERE type = 'skill_used' AND json_extract(data, '$.skillName') = ?
ORDER BY ts DESC LIMIT 1
```

스킬의 마지막 사용 시점이 없거나 `threshold`보다 이전이면 미사용으로 판정한다.

#### Scenario: 미사용 스킬 탐지

- **GIVEN** `loadSkills()`가 `['ts-init', 'docker-setup', 'test-runner']`를 반환하고, `events` 테이블에서 `ts-init`만 최근 30일 내 사용 이력이 있을 때
- **WHEN** `findStaleSkills(30)`을 호출하면
- **THEN** `['docker-setup', 'test-runner']`를 반환한다

#### Scenario: projectPath 지정

- **GIVEN** `loadSkills('/path/to/project')`가 프로젝트별 스킬 목록을 반환할 때
- **WHEN** `findStaleSkills(30, '/path/to/project')`를 호출하면
- **THEN** 해당 프로젝트의 스킬 중 미사용 스킬만 반환한다

#### Scenario: 모든 스킬이 활발히 사용 중

- **GIVEN** 모든 등록 스킬이 최근 30일 내 `events` 테이블에 사용 이력이 있을 때
- **WHEN** `findStaleSkills(30)`을 호출하면
- **THEN** 빈 배열 `[]`을 반환한다

#### Scenario: DB 접근 실패

- **GIVEN** `self-gen.db` 파일이 존재하지 않거나 접근할 수 없을 때
- **WHEN** `findStaleSkills(30)`을 호출하면
- **THEN** 빈 배열 `[]`을 반환한다

---

### REQ-SE-106: 피드백 → AI 프롬프트 주입

`getFeedbackSummary()`의 반환값은 AI 분석 프롬프트(`analyze.md`)에 다음 형식으로 주입되어야 한다(SHALL):

```markdown
## 사용자 피드백 이력 (선호도 학습용)

채택률: {{rate}}% ({{total}}건 중 {{acceptedCount}}건 채택)

최근 거부된 제안 (유사 제안 피할 것):
{{recentRejections}}

최근 채택된 제안 (이런 유형의 제안 선호):
{{recentAcceptances}}
```

**비고**: 프롬프트 주입 자체는 `ai-analyzer` 모듈이 담당하며, `feedback-tracker`는 데이터 제공만 책임진다.

#### Scenario: 피드백 컨텍스트 주입

- **GIVEN** `getFeedbackSummary()`가 `{ rate: 0.7, total: 10, acceptedCount: 7, recentRejections: ['X'], recentAcceptances: ['Y'] }`를 반환할 때
- **WHEN** AI 분석 프롬프트가 생성되면
- **THEN** 프롬프트에 채택률, 거부 이력, 채택 이력이 포함된다

#### Scenario: 피드백 없음

- **GIVEN** `getFeedbackSummary()`가 `null`을 반환할 때
- **WHEN** AI 분석 프롬프트가 생성되면
- **THEN** 피드백 섹션은 생략된다

---

## 비기능 요구사항

### 성능

- `recordFeedback()`: 10ms 이내 (SHOULD) — 단일 INSERT 쿼리
- `getFeedbackSummary()`: 50ms 이내 (SHOULD) — SELECT + JavaScript 집계
- `calcSkillUsageRate()`, `calcRuleEffectiveness()`: 100ms 이내 (SHOULD) — SQL COUNT 쿼리

### 데이터 무결성

- `feedback` 테이블은 INSERT-only로 기록해야 한다(SHALL) — UPDATE/DELETE 금지
- 모든 레코드에 `v` 컬럼으로 스키마 버전(기본값 1)을 포함해야 한다(SHALL)
- SQLite 트랜잭션으로 데이터 무결성을 보장해야 한다(SHALL)

### 에러 처리

- 모든 함수는 에러 발생 시 예외를 throw하지 않아야 한다(SHALL)
- `recordFeedback`: 예외 발생 시 조용히 실패 (try-catch)
- `getFeedbackSummary`: 예외 발생 시 `null` 반환
- `calcSkillUsageRate`: 예외 발생 시 `null` 반환
- `calcRuleEffectiveness`: 예외 발생 시 `null` 반환
- `findStaleSkills`: 예외 발생 시 빈 배열 `[]` 반환

---

## 제약사항

- `better-sqlite3` 외 추가 npm 패키지 사용 금지
- ES Modules 형식 (`.mjs` 확장자)
- `events` 테이블 읽기는 `lib/db.mjs`의 `getDb()`를 통해 직접 SQL 실행
- `feedback` 테이블 쓰기/읽기는 `lib/db.mjs`의 `getDb()`를 통해 직접 SQL 실행
- `loadSkills()`는 `lib/skill-matcher.mjs`에서 import
- 날짜 비교는 JavaScript `Date.now()` 기반 계산 후 ISO 8601 문자열 변환 사용

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| 피드백(feedback) | 제안에 대한 사용자 행동 기록 (채택/거부/무시) |
| `feedback` 테이블 | 피드백 레코드를 저장하는 SQLite 테이블 (`self-gen.db`) |
| `events` 테이블 | 이벤트 로그를 저장하는 SQLite 테이블 (`self-gen.db`) |
| 채택률(rate) | `acceptedCount / total` 비율 (0~1) |
| 스킬 사용률(skillUsageRate) | `skill_used` 이벤트 수 / `skill_created` 이벤트 수 |
| 규칙 효과성(ruleEffectiveness) | `{ totalErrors, recentErrors }` — 전체 에러 수와 최근 7일 에러 수 |
| 미사용 스킬(staleSkills) | 지정 기간 내 `events` 테이블에 사용 이력이 없는 스킬 이름 배열 |
| 내부 함수 | export하지 않고 모듈 내부에서만 사용되는 함수 |
