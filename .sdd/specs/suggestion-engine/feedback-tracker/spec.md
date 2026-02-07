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

> 제안 채택/거부 피드백을 `feedback` 테이블에 기록하고, SQL 집계로 요약을 생성하며, `events` 테이블 기반으로 스킬 사용률/규칙 효과성/미사용 스킬 등 효과성 메트릭을 산출하는 유틸리티 모듈. AI 분석 프롬프트에 피드백 컨텍스트를 주입하여 제안 품질을 자체 개선한다. DB 접근은 `lib/db.mjs`를 통한다.

---

## 개요

`lib/feedback-tracker.mjs`는 제안 적용/거부 이력을 `feedback` 테이블에 기록하고, SQL 집계를 통해 AI 분석 시 주입할 피드백 요약을 생성한다. 정적 임계값 대신 피드백 데이터를 AI 프롬프트 컨텍스트로 전달하여 Claude가 사용자 선호도를 직접 학습하게 한다.

### 대상 파일/테이블

| 대상 | 역할 |
|------|------|
| `lib/feedback-tracker.mjs` | 피드백 기록, 요약, 효과성 메트릭 |
| `feedback` 테이블 (`self-gen.db`) | 피드백 데이터 저장소 |
| `events` 테이블 (`self-gen.db`) | 스킬 사용/에러 이벤트 조회 대상 |

### 주요 함수

| 함수 | 설명 |
|------|------|
| `recordFeedback(id, action, meta)` | 피드백 엔트리 기록 |
| `getFeedbackSummary()` | AI 프롬프트 주입용 요약 생성 |
| `calcSkillUsageRate()` | 스킬 실제 사용률 계산 |
| `calcRuleEffectiveness()` | 규칙 추가 후 에러 재발 여부 측정 |
| `findStaleSkills(days)` | 장기 미사용 스킬 탐지 |

---

## 요구사항

### REQ-SE-101: 피드백 기록

시스템은 `recordFeedback(suggestionId, action, details)` 함수를 통해 피드백을 `feedback` 테이블에 INSERT해야 한다(SHALL). `action`은 `'accepted'`, `'rejected'`, `'dismissed'` 중 하나여야 한다(SHALL).

```sql
INSERT INTO feedback (v, ts, suggestion_id, action, suggestion_type, summary)
VALUES (1, ?, ?, ?, ?, ?)
```

#### Scenario: 채택 피드백 기록

- **GIVEN** 제안 ID `sg-001`이 스킬 유형으로 적용되었을 때
- **WHEN** `recordFeedback('sg-001', 'accepted', { suggestionType: 'skill', summary: 'TS 초기화 스킬' })`이 호출되면
- **THEN** `feedback` 테이블에 `{ v: 1, ts: '<ISO>', suggestion_id: 'sg-001', action: 'accepted', suggestion_type: 'skill', summary: 'TS 초기화 스킬' }` 레코드가 INSERT된다

#### Scenario: 거부 피드백 기록

- **GIVEN** 제안 ID `sg-002`가 거부되었을 때
- **WHEN** `recordFeedback('sg-002', 'rejected', { suggestionType: 'unknown' })`이 호출되면
- **THEN** `feedback` 테이블에 `{ v: 1, ts: '<ISO>', suggestion_id: 'sg-002', action: 'rejected', suggestion_type: 'unknown' }` 레코드가 INSERT된다

---

### REQ-SE-102: 피드백 요약 생성

시스템은 `getFeedbackSummary()` 함수를 통해 SQL 집계로 AI 분석 프롬프트에 주입할 피드백 요약을 반환해야 한다(SHALL). 반환 객체는 `total`, `acceptedCount`, `rejectedCount`, `rate`, `recentRejections`, `recentAcceptances`, `skillUsageRate`, `ruleEffectiveness`, `staleSkills` 필드를 포함해야 한다(SHALL).

```sql
SELECT
  COUNT(*) as total,
  SUM(CASE WHEN action = 'accepted' THEN 1 ELSE 0 END) as accepted_count,
  SUM(CASE WHEN action IN ('rejected', 'dismissed') THEN 1 ELSE 0 END) as rejected_count
FROM feedback
```

#### Scenario: 정상 요약 생성

- **GIVEN** `feedback` 테이블에 채택 7건, 거부 3건이 기록되어 있을 때
- **WHEN** `getFeedbackSummary()`를 호출하면
- **THEN** `{ total: 10, acceptedCount: 7, rejectedCount: 3, rate: 0.7, recentRejections: [...], recentAcceptances: [...], skillUsageRate: <number|null>, ruleEffectiveness: <object|null>, staleSkills: [...] }`를 반환한다

**비고**: `rejectedCount`는 `action = 'rejected'` 또는 `action = 'dismissed'`인 레코드를 모두 포함하여 카운트해야 한다(SHALL).

#### Scenario: 피드백 레코드 없음

- **GIVEN** `feedback` 테이블이 비어 있을 때
- **WHEN** `getFeedbackSummary()`를 호출하면
- **THEN** `null`을 반환한다

#### Scenario: 최근 이력 제한

- **GIVEN** `feedback` 테이블에 채택 15건, 거부 12건이 기록되어 있을 때
- **WHEN** `getFeedbackSummary()`를 호출하면
- **THEN** `recentRejections`는 최근 10건만, `recentAcceptances`는 최근 10건만 포함한다 (`ORDER BY ts DESC LIMIT 10`)

---

### REQ-SE-103: 스킬 사용률 계산

시스템은 `calcSkillUsageRate()` 함수를 통해 생성된 스킬의 실제 사용률을 계산해야 한다(SHALL). `events` 테이블에서 SQL COUNT로 `type = 'skill_used'` 이벤트 수를 `type = 'skill_created'` 이벤트 수로 나눈 비율을 반환해야 한다(SHALL).

```sql
SELECT COUNT(*) FROM events WHERE type = 'skill_used'
SELECT COUNT(*) FROM events WHERE type = 'skill_created'
```

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

### REQ-SE-104: 규칙 효과성 측정

시스템은 `calcRuleEffectiveness()` 함수를 통해 규칙 추가 후 에러 재발 여부를 측정해야 한다(SHALL). `events` 테이블에서 SQL 집계로 `type = 'tool_error'` 이벤트의 전체 수와 최근 7일 내 발생 수를 반환해야 한다(SHALL).

```sql
SELECT
  COUNT(*) as total_errors,
  SUM(CASE WHEN ts > datetime('now', '-7 days') THEN 1 ELSE 0 END) as recent_errors
FROM events WHERE type = 'tool_error'
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

### REQ-SE-105: 미사용 스킬 탐지

시스템은 `findStaleSkills(days)` 함수를 통해 지정된 기간(기본 30일) 동안 사용되지 않은 스킬을 탐지해야 한다(SHALL). 등록된 스킬 목록에서 `events` 테이블의 `skill_used` 이벤트와 SQL 대조하여 미사용 스킬 이름 배열을 반환해야 한다(SHALL).

```sql
SELECT DISTINCT json_extract(data, '$.skillName') as skill_name
FROM events
WHERE type = 'skill_used' AND ts > datetime('now', '-' || ? || ' days')
```

#### Scenario: 미사용 스킬 탐지

- **GIVEN** 등록된 스킬 `['ts-init', 'docker-setup', 'test-runner']`이 있고, `events` 테이블에서 `ts-init`만 최근 30일 내 사용 이력이 있을 때
- **WHEN** `findStaleSkills(30)`을 호출하면
- **THEN** `['docker-setup', 'test-runner']`를 반환한다

#### Scenario: 모든 스킬이 활발히 사용 중

- **GIVEN** 모든 등록 스킬이 최근 30일 내 `events` 테이블에 사용 이력이 있을 때
- **WHEN** `findStaleSkills(30)`을 호출하면
- **THEN** 빈 배열 `[]`을 반환한다

#### Scenario: DB 접근 실패

- **GIVEN** `self-gen.db` 파일이 존재하지 않거나 접근할 수 없을 때
- **WHEN** `findStaleSkills(30)`을 호출하면
- **THEN** 빈 배열 `[]`을 반환한다

---

## 비기능 요구사항

### 성능

- `recordFeedback()`: 10ms 이내 (SHOULD) — 단일 INSERT 쿼리
- `getFeedbackSummary()`: 50ms 이내 (SHOULD) — SQL 집계 쿼리
- `calcSkillUsageRate()`, `calcRuleEffectiveness()`: 100ms 이내 (SHOULD) — SQL COUNT/SUM 쿼리

### 데이터 무결성

- `feedback` 테이블은 INSERT-only로 기록해야 한다(SHALL) — UPDATE/DELETE 금지
- 모든 레코드에 `v` 컬럼으로 스키마 버전(기본값 1)을 포함해야 한다(SHALL)
- SQLite 트랜잭션으로 데이터 무결성을 보장해야 한다(SHALL)

---

## 제약사항

- `better-sqlite3` 외 추가 npm 패키지 사용 금지
- ES Modules 형식 (`.mjs` 확장자)
- `events` 테이블 읽기는 `lib/db.mjs`의 `queryEvents()`, `getDb()` 등을 사용
- `feedback` 테이블 쓰기/읽기는 `lib/db.mjs`의 `getDb()`를 통해 직접 SQL 실행
- 에러 발생 시 함수는 `null` 또는 `[]`을 반환 (예외를 throw하지 않음)

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| 피드백(feedback) | 제안에 대한 사용자 행동 기록 (채택/거부/무시) |
| `feedback` 테이블 | 피드백 레코드를 저장하는 SQLite 테이블 (`self-gen.db`) |
| `events` 테이블 | 이벤트 로그를 저장하는 SQLite 테이블 (`self-gen.db`) |
| 채택률(rate) | `accepted_count / total` 비율 |
| 스킬 사용률(skillUsageRate) | `events` 테이블의 `skill_used` 이벤트 수 / `skill_created` 이벤트 수 |
| 규칙 효과성(ruleEffectiveness) | 규칙 추가 후 관련 에러 재발 빈도 (SQL 집계) |
| 미사용 스킬(staleSkills) | 지정 기간 내 `events` 테이블에 사용 이력이 없는 스킬 |
