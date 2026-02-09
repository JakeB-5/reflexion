---
id: subagent-context
title: "서브에이전트 컨텍스트 주입 (Subagent Context Injection)"
status: draft
created: 2026-02-07
domain: realtime-assist
depends: "data-collection/log-writer, realtime-assist/error-kb, ai-analysis/ai-analyzer"
constitution_version: "2.0.0"
---

# subagent-context

> `hooks/subagent-context.mjs` — SubagentStart 훅 이벤트를 처리하여 코드 작업 서브에이전트에 프로젝트별 에러 패턴 및 AI 분석 규칙을 주입하는 훅 스크립트. 비코드 에이전트는 건너뛴다.

---

## 개요

subagent-context는 서브에이전트 시작 시 프로젝트의 학습 데이터를 주입하여, 서브에이전트도 시스템의 축적된 지식을 활용하도록 한다. 코드 작업에 직접 관여하는 에이전트에만 컨텍스트를 주입하고, 탐색/연구 에이전트에는 불필요한 컨텍스트 주입을 방지한다.

> **v9 설계 변경**: 에러 KB 검색 시 `searchErrorKB()` (벡터 검색 포함) 대신 error_kb 테이블 직접 텍스트 쿼리를 사용한다. SubagentStart 지연을 방지하기 위함. `getCachedAnalysis()`에 project 필터를 추가하여 프로젝트별 분석 결과만 주입한다.

### 파일 위치

- 훅 스크립트: `~/.self-generation/hooks/subagent-context.mjs`
- 의존 데이터: `~/.self-generation/data/self-gen.db` (`events` 테이블, `error_kb` 테이블, `analysis_cache` 테이블)

### 훅 등록

```json
{
  "hooks": {
    "SubagentStart": [{
      "hooks": [{
        "type": "command",
        "command": "node $HOME/.self-generation/hooks/subagent-context.mjs"
      }]
    }]
  }
}
```

### 코드 에이전트 목록 (CODE_AGENTS)

```javascript
const CODE_AGENTS = [
  'executor', 'executor-low', 'executor-high',
  'architect', 'architect-medium',
  'designer', 'designer-high',
  'build-fixer', 'build-fixer-low'
];
```

---

## 요구사항

### REQ-RA-401: 코드 에이전트 필터링

시스템은 CODE_AGENTS 목록에 포함된 에이전트 타입에만 컨텍스트를 주입해야 한다(SHALL).

1. stdin의 `agent_type` 필드를 CODE_AGENTS 목록과 비교 (SHALL)
2. `agent_type`이 CODE_AGENTS 중 하나를 포함(`includes`)하면 컨텍스트 주입 진행 (SHALL)
3. CODE_AGENTS에 해당하지 않는 에이전트(explore, researcher, analyst, planner 등)는 즉시 exit 0 (SHALL)

#### Scenario RA-401-1: 코드 에이전트 시작 시

- **GIVEN** SubagentStart 이벤트의 `agent_type`이 `"executor"`
- **WHEN** subagent-context.mjs가 실행되면
- **THEN** 컨텍스트 주입 로직을 수행한다

#### Scenario RA-401-2: 비코드 에이전트 시작 시

- **GIVEN** SubagentStart 이벤트의 `agent_type`이 `"researcher"`
- **WHEN** subagent-context.mjs가 실행되면
- **THEN** 컨텍스트 주입 없이 즉시 exit 0으로 종료한다

#### Scenario RA-401-3: 복합 에이전트 타입명 매칭

- **GIVEN** SubagentStart 이벤트의 `agent_type`이 `"oh-my-claudecode:executor-high"`
- **WHEN** subagent-context.mjs가 실행되면
- **THEN** `"executor-high"`가 CODE_AGENTS에 포함되므로 컨텍스트 주입을 수행한다

---

### REQ-RA-402: 프로젝트별 최근 에러 패턴 주입 — 텍스트 직접 쿼리

시스템은 해당 프로젝트의 최근 에러 3건과 error_kb 해결 이력을 서브에이전트에 주입해야 한다(SHALL).

> **v9 설계**: `searchErrorKB()` (벡터 검색 포함) 대신 error_kb 테이블 직접 텍스트 쿼리를 사용한다. 벡터 검색 ~5ms×3건 + 임베딩 오버헤드를 방지하여 SubagentStart 지연을 최소화.

1. `queryEvents()`로 현재 프로젝트의 `tool_error` 타입 이벤트를 최근 3건 조회 (SHALL):
   ```javascript
   const projectErrors = queryEvents({ type: 'tool_error', projectPath: projectDir, limit: 3 });
   ```
2. 각 에러에 대해 `error_kb` 테이블에서 **정확 텍스트 매치**로 해결 이력을 직접 조회 (SHALL):
   ```sql
   SELECT resolution FROM error_kb
   WHERE error_normalized = ? AND resolution IS NOT NULL
   ORDER BY use_count DESC LIMIT 1
   ```
3. 에러 패턴과 도구명을 포함하여 표시 (SHALL):
   - `이 프로젝트의 최근 에러 패턴:`
   - `- ${err.error} (${err.tool})`
   - `  해결: ${JSON.stringify(kb.resolution).slice(0, 150)}` (해결 이력이 있는 경우)
4. 각 해결 방법 텍스트는 150자로 제한 (SHALL)
5. 에러가 없으면 에러 패턴 섹션을 출력하지 않는다(SHALL)

#### Scenario RA-402-1: 프로젝트에 에러 이력이 있을 때

- **GIVEN** `events` 테이블에 현재 프로젝트의 `tool_error` 엔트리 5건이 있고, 그 중 2건에 `error_kb` 정확 매치 해결 이력이 존재
- **WHEN** 코드 에이전트가 시작되면
- **THEN** `additionalContext`에 "이 프로젝트의 최근 에러 패턴:" 섹션과 에러 3건 + 해결 이력이 포함된다

#### Scenario RA-402-2: 프로젝트에 에러 이력이 없을 때

- **GIVEN** `events` 테이블에 현재 프로젝트의 에러 기록이 없을 때
- **WHEN** 코드 에이전트가 시작되면
- **THEN** 에러 패턴 섹션은 출력하지 않는다

---

### REQ-RA-403: AI 분석 규칙 주입 — 프로젝트 필터

시스템은 캐시된 AI 분석 결과에서 현재 프로젝트에 관련된 CLAUDE.md 규칙을 서브에이전트에 주입해야 한다(SHALL).

1. `getCachedAnalysis(48, project)`로 48시간 이내의 프로젝트별 캐시를 조회 (SHALL)
2. 캐시 내 `suggestions` 중 `type === 'claude_md'`이고 프로젝트가 일치하거나 전역(`project` 없음 또는 `null`)인 항목을 필터링 (SHALL):
   ```javascript
   const rules = analysis.suggestions
     .filter(s => s.type === 'claude_md' && (!s.project || s.project === project))
     .slice(0, 3);
   ```
3. 최대 3건까지 선택하여 "적용할 프로젝트 규칙:" 섹션으로 주입 (SHALL)
4. 각 규칙의 `rule` 또는 `summary` 필드를 사용 (SHALL)

#### Scenario RA-403-1: AI 분석 규칙이 있을 때

- **GIVEN** `getCachedAnalysis(48, project)`가 현재 프로젝트에 대한 `claude_md` 타입 제안 2건을 포함한 캐시를 반환
- **WHEN** 코드 에이전트가 시작되면
- **THEN** `additionalContext`에 "적용할 프로젝트 규칙:" 섹션과 2건의 규칙이 포함된다

#### Scenario RA-403-2: AI 분석 캐시가 만료되었을 때

- **GIVEN** `getCachedAnalysis(48, project)`가 `null`을 반환
- **WHEN** 코드 에이전트가 시작되면
- **THEN** AI 규칙 섹션은 출력하지 않는다

#### Scenario RA-403-3: 전역 규칙과 프로젝트 규칙 혼합

- **GIVEN** 캐시에 전역(`project: null`) 규칙 1건과 현재 프로젝트 규칙 2건이 존재
- **WHEN** 코드 에이전트가 시작되면
- **THEN** 3건 모두 "적용할 프로젝트 규칙:" 섹션에 포함된다

---

### REQ-RA-404: 컨텍스트 크기 제한

시스템은 주입하는 전체 컨텍스트를 최대 500자로 제한해야 한다(SHALL).

1. 에러 패턴 + AI 규칙의 전체 텍스트를 `\n`으로 결합한 후 `.slice(0, 500)`으로 절단 (SHALL)
2. 컨텍스트가 비어있으면 stdout 출력 없이 종료 (SHALL)

#### Scenario RA-404-1: 컨텍스트 500자 초과 시

- **GIVEN** 에러 3건 + AI 규칙 3건의 합산 텍스트가 700자
- **WHEN** subagent-context.mjs가 실행되면
- **THEN** 500자로 절단된 `additionalContext`를 출력한다

#### Scenario RA-404-2: 주입할 컨텍스트가 없을 때

- **GIVEN** 프로젝트 에러 이력도 없고 AI 분석 캐시도 없을 때
- **WHEN** 코드 에이전트가 시작되면
- **THEN** stdout 출력 없이 exit 0으로 종료한다

---

### REQ-RA-405: 출력 형식

시스템은 컨텍스트를 SubagentStart 훅의 표준 출력 형식으로 전달해야 한다(SHALL).

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SubagentStart",
    "additionalContext": "컨텍스트 내용..."
  }
}
```

> **주의**: `hookEventName`은 `"SubagentStart"`이어야 한다 (SHALL).

#### Scenario RA-405-1: 정상 출력

- **GIVEN** 주입할 에러 패턴 + AI 규칙 컨텍스트가 존재
- **WHEN** subagent-context.mjs가 실행되면
- **THEN** `hookEventName: "SubagentStart"`와 `additionalContext`를 포함한 JSON이 stdout으로 출력된다

---

### REQ-RA-406: 비차단 실행 보장

훅 스크립트는 어떤 상황에서도 exit 0으로 종료해야 한다(SHALL).

1. 전체 로직을 try-catch로 감싸야 한다(SHALL)
2. 예외 발생 시 `process.exit(0)`으로 종료 (SHALL)

#### Scenario RA-406-1: 의존 모듈 로드 실패 시

- **GIVEN** `ai-analyzer.mjs` 모듈이 존재하지 않아 import 실패
- **WHEN** subagent-context.mjs가 실행되면
- **THEN** exit code 0으로 정상 종료한다

---

### REQ-RA-407: isEnabled 체크

시스템은 `isEnabled()` 함수로 시스템 활성화 상태를 확인하고, 비활성화 시 즉시 종료해야 한다(SHALL).

#### Scenario RA-407-1: 시스템 비활성화 시

- **GIVEN** `config.json`에서 `enabled: false`로 설정
- **WHEN** subagent-context.mjs가 실행되면
- **THEN** 컨텍스트를 주입하지 않고 즉시 exit 0으로 종료한다

---

## 비기능 요구사항

### 성능

- 훅 실행 시간은 2초 이내여야 한다(SHALL)
- 벡터 검색을 사용하지 않고 텍스트 매칭만 사용하여 SubagentStart 지연을 최소화한다(SHALL)
- SQL 쿼리는 인덱스를 활용하여 대용량 데이터에서도 빠르게 동작해야 한다(SHOULD)

---

## 제약사항

- `better-sqlite3` 패키지를 사용하여 SQLite 접근 (SHALL)
- `searchErrorKB()`는 사용하지 않음 — 벡터 검색 회피를 위해 error_kb 테이블 직접 쿼리 (SHALL)
- `ai-analyzer.mjs`의 `getCachedAnalysis(hours, project)` 함수를 import하여 사용 (SHALL) — project 파라미터 필수
- `db.mjs`의 `queryEvents()`, `getDb()`, `getProjectName()`, `getProjectPath()`, `readStdin()`, `isEnabled()` 함수를 import하여 사용 (SHALL)
- CODE_AGENTS 목록은 모듈 상단에 상수로 정의 (SHALL)

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| SubagentStart | Claude Code에서 서브에이전트가 시작될 때 발생하는 훅 이벤트 |
| CODE_AGENTS | 코드 작업에 직접 관여하는 에이전트 타입 목록 |
| 비코드 에이전트 | 탐색, 연구, 분석 등 코드를 직접 수정하지 않는 에이전트 |
| analysis_cache 테이블 | AI 분석 결과를 캐시하는 SQLite 테이블 |
| 컨텍스트 주입 | 훅이 stdout으로 출력한 additionalContext가 서브에이전트에 전달되는 메커니즘 |
| 텍스트 직접 쿼리 | 벡터 검색 없이 SQL `=` 연산자로 error_kb 테이블을 직접 조회하는 방식 |
