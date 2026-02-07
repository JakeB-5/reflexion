---
id: ai-analyzer
title: "AI 분석 실행 모듈"
status: draft
created: 2026-02-07
domain: ai-analysis
depends: "data-collection/log-writer, suggestion-engine/feedback-tracker, realtime-assist/skill-matcher"
constitution_version: "2.0.0"
---

# AI 분석 실행 모듈 (ai-analyzer)

> `claude --print` CLI를 래핑하여 수집된 이벤트 데이터(`events` 테이블)를 AI로 의미 분석하고, 결과를 `analysis_cache` 테이블에 저장하며, 프롬프트 템플릿(`prompts/analyze.md`)을 관리하는 핵심 분석 모듈. DB 접근은 `lib/db.mjs`를 통한다.

---

## 요구사항

### REQ-AA-001: 동기 AI 분석 실행 (runAnalysis)

시스템은 `claude --print`를 동기적으로 실행하여 로그 데이터를 분석하고 JSON 결과를 반환해야 한다(SHALL). 분석 대상 프롬프트가 5개 미만이면 분석을 생략해야 한다(SHALL).

#### Scenario: 충분한 데이터로 분석 실행

- **GIVEN** `events` 테이블에 최근 7일간 프롬프트 10개 이상이 기록되어 있다
- **WHEN** `runAnalysis({ days: 7, project: 'my-app' })`을 호출한다
- **THEN** `claude --print`가 실행되고, clusters/workflows/errorPatterns/suggestions/skill_descriptions을 포함한 JSON 결과가 반환된다

#### Scenario: 데이터 부족 시 분석 생략

- **GIVEN** `events` 테이블에 최근 7일간 프롬프트가 3개만 기록되어 있다
- **WHEN** `runAnalysis({ days: 7 })`을 호출한다
- **THEN** `{ suggestions: [], reason: 'insufficient_data' }`가 반환되고 `claude --print`는 실행되지 않는다

#### Scenario: claude --print 실행 실패

- **GIVEN** `claude` CLI가 설치되지 않았거나 네트워크 오류가 발생한다
- **WHEN** `runAnalysis()`을 호출한다
- **THEN** `{ suggestions: [], error: '<에러 메시지>' }`가 반환되고 예외가 전파되지 않는다(SHALL NOT)

---

### REQ-AA-002: 비동기 AI 분석 실행 (runAnalysisAsync)

시스템은 AI 분석을 백그라운드 프로세스로 실행할 수 있어야 한다(SHALL). 부모 프로세스와 분리(`detached`, `unref`)되어 호출자를 블로킹하지 않아야 한다(SHALL).

#### Scenario: SessionEnd에서 비동기 분석 트리거

- **GIVEN** SessionEnd 훅이 세션 요약을 기록 완료했다
- **WHEN** `runAnalysisAsync({ days: 7, project: 'my-app' })`을 호출한다
- **THEN** `node bin/analyze.mjs --days 7 --project my-app`이 분리된 자식 프로세스로 실행되고, 호출자는 즉시 반환된다

#### Scenario: 부모 프로세스 종료 후에도 분석 계속

- **GIVEN** `runAnalysisAsync()`로 백그라운드 분석이 시작되었다
- **WHEN** 부모 프로세스(SessionEnd 훅)가 종료된다
- **THEN** 자식 프로세스는 분석을 계속 수행하고 결과를 캐시에 저장한다

---

### REQ-AA-003: 프롬프트 빌드 (buildPrompt)

시스템은 `prompts/analyze.md` 템플릿 파일을 로드하고, 로그 데이터/피드백 이력/기존 스킬 목록/제안 효과 메트릭을 주입하여 완성된 프롬프트를 생성해야 한다(SHALL). 피드백 데이터는 `feedback` 테이블에서 조회한다.

#### Scenario: 모든 데이터가 존재할 때 프롬프트 빌드

- **GIVEN** `prompts/analyze.md` 템플릿이 존재하고, `feedback` 테이블에 피드백 이력이 있고 기존 스킬이 등록되어 있다
- **WHEN** `buildPrompt(logSummary, 7, 'my-app')`을 호출한다
- **THEN** 템플릿의 `{{days}}`, `{{project}}`, `{{log_data}}`, `{{feedback_history}}`, `{{existing_skills}}`, `{{outcome_metrics}}` 플레이스홀더가 모두 실제 데이터로 치환된 문자열이 반환된다

#### Scenario: 피드백 이력이 없는 첫 분석

- **GIVEN** `feedback` 테이블이 비어 있고 등록된 스킬이 없다
- **WHEN** `buildPrompt(logSummary, 7, null)`을 호출한다
- **THEN** `{{feedback_history}}`는 `'피드백 이력 없음 (첫 분석)'`으로, `{{existing_skills}}`는 `'등록된 스킬 없음'`으로, `{{project}}`는 `'all'`로 치환된다

**비고**: buildPrompt 내에서 `loadSkills()`를 인자 없이 호출하므로 전역 스킬(`~/.claude/commands/`)만 로드됨. 프로젝트별 스킬은 포함되지 않음.

---

### REQ-AA-004: JSON 응답 추출 (extractJSON)

시스템은 Claude 응답에서 JSON 부분만 안전하게 추출해야 한다(SHALL). ````json` 코드 블록 형태와 순수 JSON 형태를 모두 지원해야 한다(SHALL).

#### Scenario: 코드 블록 내 JSON 추출

- **GIVEN** Claude 응답이 ````json\n{ "clusters": [...] }\n````  형태이다
- **WHEN** `extractJSON(response)`을 호출한다
- **THEN** 코드 블록 안의 JSON 문자열만 추출되어 반환된다

#### Scenario: 순수 JSON 응답 추출

- **GIVEN** Claude 응답이 `{ "clusters": [...] }` 형태로 코드 블록 없이 반환되었다
- **WHEN** `extractJSON(response)`을 호출한다
- **THEN** 전체 텍스트에서 `{...}` 패턴을 찾아 JSON 문자열이 반환된다

---

### REQ-AA-005: 분석 캐시 관리 (getCachedAnalysis)

시스템은 AI 분석 결과를 `analysis_cache` 테이블에 캐시하고, TTL 기반으로 조회해야 한다(SHALL). 캐시가 만료되었거나 존재하지 않으면 `null`을 반환해야 한다(SHALL).

```sql
SELECT * FROM analysis_cache
WHERE ts > datetime('now', '-' || ? || ' hours')
ORDER BY ts DESC LIMIT 1
```

캐시 저장 시 `INSERT INTO analysis_cache (ts, project, days, analysis) VALUES (?, ?, ?, json(?))` 쿼리를 사용한다.

#### Scenario: 유효한 캐시 조회

- **GIVEN** `analysis_cache` 테이블에 2시간 전 분석 결과가 저장되어 있다
- **WHEN** `getCachedAnalysis(24)`를 호출한다
- **THEN** 캐시된 분석 결과(`analysis` 객체)가 반환된다

#### Scenario: 캐시 만료

- **GIVEN** `analysis_cache` 테이블에 30시간 전 분석 결과가 저장되어 있다
- **WHEN** `getCachedAnalysis(24)`를 호출한다
- **THEN** `null`이 반환된다

#### Scenario: 캐시 레코드 없음

- **GIVEN** `analysis_cache` 테이블이 비어 있다
- **WHEN** `getCachedAnalysis(24)`를 호출한다
- **THEN** `null`이 반환된다

---

### REQ-AA-006: 로그 요약 (summarizeForPrompt)

시스템은 `events` 테이블에서 SQL 집계를 활용하여 토큰 절약된 요약을 생성해야 한다(SHALL). 프롬프트는 최근 100개로 제한하고(SHOULD), 세션별 도구 시퀀스를 집계해야 한다(SHALL).

```sql
SELECT * FROM events
WHERE project_path = ? AND ts > ? ORDER BY ts DESC LIMIT ?
```

#### Scenario: 대량 로그 요약

- **GIVEN** `events` 테이블에 500개의 프롬프트, 2000개의 도구 사용, 15개의 에러가 기록되어 있다
- **WHEN** `summarizeForPrompt(projectPath, days)`를 호출한다
- **THEN** 프롬프트는 최근 100개만 포함되고, 도구 사용은 세션별 시퀀스 문자열(`'Grep → Read → Edit'`)로 요약되며, 에러는 `{ tool, error, raw }` 형태로 축약된다

#### Scenario: 프롬프트 필드 추출

- **GIVEN** `events` 테이블에 `ts`, `text`, `project`, `session_id` 등 다양한 필드가 있다
- **WHEN** `summarizeForPrompt(projectPath, days)`를 호출한다
- **THEN** 다음 구조의 요약 객체를 반환한다:
  ```javascript
  {
    prompts: [{ ts, text, project }],           // 프롬프트 배열 (최근 100개로 제한)
    toolSequences: string[],                    // 세션별 '도구→도구' 시퀀스 문자열
    errors: [{ tool, error, raw }],             // 축약된 에러 배열
    sessionCount: number,                       // 분석 대상 세션 수
    toolTotal: number                           // 총 도구 호출 횟수
  }
  ```

---

## 비기능 요구사항

### 성능

- `runAnalysis()` 동기 실행 시 `claude --print` 응답 시간에 의존하며, 최대 60초 이내에 완료되어야 한다(SHOULD)
- `getCachedAnalysis()` 캐시 조회는 100ms 이내에 완료되어야 한다(SHALL) — SQLite 인덱스 활용
- `summarizeForPrompt()`는 10,000개 레코드 기준 500ms 이내에 완료되어야 한다(SHOULD) — SQL 집계 활용

### 안정성

- AI 분석 실패 시 빈 결과를 반환하고, 예외를 전파하지 않아야 한다(SHALL)
- DB 읽기/쓰기 실패 시 `null`을 반환하고, 시스템 동작에 영향을 주지 않아야 한다(SHALL)

---

## 제약사항

- `claude` CLI가 시스템에 설치되어 있어야 한다 (Claude Code 사용자 전제)
- `better-sqlite3` 외 추가 npm 패키지 없이 Node.js 내장 모듈만 사용해야 한다(SHALL)
- `maxBuffer`는 10MB로 설정하여 대용량 응답을 처리해야 한다(SHALL)

---

## AI 분석 출력 스키마

| 필드 | 타입 | 설명 |
|------|------|------|
| `clusters` | array | 의미적으로 유사한 프롬프트 그룹 |
| `workflows` | array | 반복되는 도구 시퀀스 패턴 |
| `errorPatterns` | array | 반복 에러 패턴 및 방지 규칙 |
| `suggestions` | array | 개선 제안 (skill/claude_md/hook) |
| `skill_descriptions` | object | 스킬 이름 → 설명/키워드 매핑 (벡터 임베딩 생성용) |

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| `claude --print` | Claude CLI의 비대화형 모드. 프롬프트를 받아 텍스트 응답만 반환 |
| TTL | Time To Live. 캐시 유효 기간 (시간 단위) |
| `analysis_cache` 테이블 | AI 분석 결과를 저장하는 SQLite 테이블 (`self-gen.db`) |
| `events` 테이블 | 전역 이벤트를 저장하는 SQLite 테이블 (`self-gen.db`) |
| `db.mjs` | SQLite DB 접근 유틸리티 모듈 (`lib/db.mjs`) |
| prompts/analyze.md | AI 분석에 사용되는 프롬프트 템플릿 파일 |
