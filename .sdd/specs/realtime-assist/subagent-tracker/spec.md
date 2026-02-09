---
id: subagent-tracker
title: "서브에이전트 추적기 (Subagent Tracker)"
status: draft
created: 2026-02-07
domain: realtime-assist
depends: data-collection/log-writer
constitution_version: "2.0.0"
---

# subagent-tracker

> `hooks/subagent-tracker.mjs` — SubagentStop 훅 이벤트를 처리하여 서브에이전트 이벤트를 `events` 테이블에 기록하는 훅 스크립트. 에이전트 타입별 사용 통계는 SQL 집계 쿼리로 조회한다.

---

## 개요

subagent-tracker는 Claude Code의 SubagentStop 이벤트를 수신하여 서브에이전트의 실행 정보를 추적한다. 이 데이터는 AI 배치 분석에서 에이전트 사용 최적화 제안의 근거가 된다. 별도의 통계 파일 없이 `events` 테이블에 대한 SQL 집계로 사용 통계를 제공한다.

> **v9 변경**: `success` 필드 제거. SubagentStop 공식 stdin에 `error` 필드가 없어 항상 `true`였으므로, dead code 방지를 위해 삭제. 9장 스키마 `{ agentId, agentType }`과 일치. 향후 API에 실패 정보가 추가되면 `agent_transcript_path` 파싱으로 정확한 판정 구현.

### 파일 위치

- 훅 스크립트: `~/.self-generation/hooks/subagent-tracker.mjs`
- 이벤트 저장: `~/.self-generation/data/self-gen.db` (`events` 테이블)

### 훅 등록

```json
{
  "hooks": {
    "SubagentStop": [{
      "hooks": [{
        "type": "command",
        "command": "node $HOME/.self-generation/hooks/subagent-tracker.mjs"
      }]
    }]
  }
}
```

### SubagentStop stdin 필드

SubagentStop 이벤트의 stdin에 제공되는 필드:

| 필드 | 설명 |
|------|------|
| `agent_id` | 서브에이전트 고유 ID |
| `agent_type` | 서브에이전트 타입 (executor, architect 등) |
| `session_id` | 세션 ID |
| `cwd` | 작업 디렉토리 |
| `agent_transcript_path` | 서브에이전트 트랜스크립트 파일 경로 |

> **참고**: SubagentStop stdin에는 `error` 또는 `success` 필드가 없다. 성공/실패 판정이 불가하므로 해당 정보를 기록하지 않는다.

### SubagentStop 이벤트 데이터 구조

`events` 테이블에 `type = 'subagent_stop'`으로 저장되며, `data` 컬럼에 JSON으로 상세 정보를 포함한다:

```jsonc
// events 테이블 행
{
  "type": "subagent_stop",
  "ts": "2026-02-07T10:00:00Z",
  "session_id": "session-abc",
  "project": "my-project",
  "project_path": "/Users/dev/my-project",
  "data": {
    "agentId": "agent-123",
    "agentType": "executor"
    // v9: success 필드 제거 — SubagentStop API에 실패 정보 없음
  }
}
```

### 통계 집계 쿼리

별도의 `subagent-stats.jsonl` 파일 대신, SQL 집계 쿼리로 사용 통계를 조회한다:

```sql
-- Agent type별 사용 통계
SELECT json_extract(data, '$.agentType') AS agent_type,
       COUNT(*) AS total
FROM events
WHERE type = 'subagent_stop'
GROUP BY agent_type

-- 특정 agent type의 최근 N건 사용 이력
SELECT json_extract(data, '$.agentType') AS agent_type,
       ts, session_id
FROM events
WHERE type = 'subagent_stop'
  AND json_extract(data, '$.agentType') LIKE ?
ORDER BY ts DESC
LIMIT ?
```

---

## 요구사항

### REQ-RA-201: SubagentStop 이벤트 기록

시스템은 SubagentStop 훅 이벤트를 수신하여 `events` 테이블에 기록해야 한다(SHALL).

기록 필드:
1. `type: "subagent_stop"` — 이벤트 타입 (SHALL)
2. `ts` — ISO 8601 타임스탬프 (SHALL)
3. `sessionId` — stdin의 `session_id` (SHALL)
4. `project` — `getProjectName(getProjectPath(input.cwd))` (SHALL)
5. `projectPath` — `getProjectPath(input.cwd)` (SHALL)
6. `agentId` — stdin의 `agent_id` (SHALL)
7. `agentType` — stdin의 `agent_type` (SHALL)

> **v9 주의**: `success` 필드는 기록하지 않는다(SHALL NOT). SubagentStop API가 성공/실패 정보를 제공하지 않기 때문이다.

`db.mjs`의 `insertEvent()` 함수를 사용하여 기록한다(SHALL).

#### Scenario RA-201-1: 정상적인 SubagentStop 이벤트 기록

- **GIVEN** SubagentStop 이벤트가 stdin으로 `{ "session_id": "s1", "cwd": "/project", "agent_id": "a1", "agent_type": "executor" }` 제공
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** `events` 테이블에 `type: "subagent_stop"`, `data: { agentId: "a1", agentType: "executor" }` 행이 INSERT된다 (success 필드 없음)

#### Scenario RA-201-2: stdin 필드 일부 누락 시

- **GIVEN** stdin에 `agent_type` 필드가 누락된 입력
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** 가능한 필드만으로 기록하고 정상 종료한다 (exit 0)

---

### REQ-RA-202: 서브에이전트 사용 통계 — SQL 집계

시스템은 별도의 통계 파일 없이 `events` 테이블에 대한 SQL 집계 쿼리로 에이전트 타입별 사용 통계를 제공해야 한다(SHALL).

1. SubagentStop 이벤트를 `agentId`, `agentType` 필드로 기록한다(SHALL)
2. 통계 조회는 `events` 테이블에 대한 SQL 집계로 수행한다(SHALL):
   ```sql
   SELECT json_extract(data, '$.agentType') AS agent_type,
          COUNT(*) AS total
   FROM events WHERE type = 'subagent_stop'
   GROUP BY agent_type
   ```
3. 별도의 `subagent-stats.jsonl` 파일은 사용하지 않는다(SHALL)
4. 성공/실패율 통계는 현재 SubagentStop API 한계로 제공하지 않는다(SHALL NOT)

> **향후 계획**: SubagentStop API에 실패 정보가 추가되면, `agent_transcript_path` 파싱으로 정확한 성공/실패 판정을 구현하고 실패율 통계를 복원할 수 있다.

#### Scenario RA-202-1: 에이전트 타입별 사용 횟수 조회

- **GIVEN** `events` 테이블에 `executor` 타입 이벤트 10건, `architect` 타입 이벤트 5건이 존재
- **WHEN** SQL 집계 쿼리로 통계를 조회하면
- **THEN** `[{ agent_type: "executor", total: 10 }, { agent_type: "architect", total: 5 }]` 결과를 반환한다

#### Scenario RA-202-2: 데이터가 없을 때

- **GIVEN** `events` 테이블에 `subagent_stop` 이벤트가 없을 때
- **WHEN** SQL 집계 쿼리로 통계를 조회하면
- **THEN** 빈 결과를 반환한다

---

### REQ-RA-203: 비차단 실행 보장

훅 스크립트는 어떤 상황에서도 exit 0으로 종료해야 한다(SHALL).

1. 전체 로직을 try-catch로 감싸야 한다(SHALL)
2. 예외 발생 시 조용히 `process.exit(0)`으로 종료 (SHALL)
3. stdin 파싱 실패, DB 쓰기 실패 등 모든 에러를 흡수 (SHALL)
4. stdout 출력 없음 — 이 훅은 컨텍스트 주입을 하지 않고 데이터 수집만 수행 (SHALL)

#### Scenario RA-203-1: stdin 파싱 실패 시

- **GIVEN** stdin으로 유효하지 않은 JSON이 입력됨
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** exit code 0으로 정상 종료한다

#### Scenario RA-203-2: DB 쓰기 실패 시

- **GIVEN** `self-gen.db` 파일에 쓰기 권한이 없는 환경
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** exit code 0으로 정상 종료한다

---

### REQ-RA-204: isEnabled 체크

시스템은 `isEnabled()` 함수로 시스템 활성화 상태를 확인하고, 비활성화 시 즉시 종료해야 한다(SHALL).

#### Scenario RA-204-1: 시스템 비활성화 시

- **GIVEN** `config.json`에서 `enabled: false`로 설정
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** 이벤트를 기록하지 않고 즉시 exit 0으로 종료한다

---

## 비기능 요구사항

### 성능

- 훅 실행 시간은 2초 이내여야 한다(SHALL) — Claude Code 훅 타임아웃 준수
- SQLite 동기 쓰기로 프로세스 종료 전 기록 완료 보장 (SHALL)

### 데이터 무결성

- `events` 테이블 스키마 준수 (SHALL)
- `data` 컬럼은 유효한 JSON 문자열이어야 한다(SHALL)

---

## 제약사항

- `better-sqlite3` 패키지를 사용하여 SQLite 접근 (SHALL)
- `db.mjs` 모듈의 `insertEvent()`, `readStdin()`, `isEnabled()`, `getProjectName()`, `getProjectPath()` 함수를 활용 (SHALL)
- stdout 출력 없음 — 이 훅은 컨텍스트 주입을 하지 않고 데이터 수집만 수행 (SHALL)

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| SubagentStop | Claude Code에서 서브에이전트 실행이 종료될 때 발생하는 훅 이벤트 |
| 에이전트 타입 (agentType) | 서브에이전트의 역할 구분 (executor, architect, designer 등) |
| SQL 집계 | events 테이블에 대한 GROUP BY 쿼리로 사용 통계를 산출하는 방식 |
| 비차단 (Non-blocking) | 에러 발생 시에도 exit 0으로 종료하여 Claude Code 워크플로우를 방해하지 않는 특성 |
| agent_transcript_path | 서브에이전트의 대화 기록 파일 경로 (향후 성공/실패 판정에 활용 가능) |
