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

> `hooks/subagent-tracker.mjs` — SubagentStop 훅 이벤트를 처리하여 서브에이전트 이벤트를 `events` 테이블에 기록하는 훅 스크립트. 에이전트 타입별 성능 통계는 SQL 집계 쿼리로 조회한다.

---

## 개요

subagent-tracker는 Claude Code의 SubagentStop 이벤트를 수신하여 서브에이전트의 실행 결과를 추적한다. 이 데이터는 pre-tool-guide에서 에이전트 실패율 경고에 활용되며, AI 배치 분석에서 에이전트 사용 최적화 제안의 근거가 된다. 별도의 통계 파일 없이 `events` 테이블에 대한 SQL 집계로 실시간 통계를 제공한다.

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
    "agentType": "executor",
    "success": true
  }
}
```

### 통계 집계 쿼리

별도의 `subagent-stats.jsonl` 파일 대신, SQL 집계 쿼리로 통계를 조회한다:

```sql
-- Agent type별 전체 통계
SELECT json_extract(data, '$.agentType') AS agent_type,
       COUNT(*) AS total,
       SUM(CASE WHEN json_extract(data, '$.success') = 1 THEN 1 ELSE 0 END) AS successes,
       SUM(CASE WHEN json_extract(data, '$.success') = 0 THEN 1 ELSE 0 END) AS failures
FROM events
WHERE type = 'subagent_stop'
GROUP BY agent_type

-- 특정 agent type의 최근 N건 실패율
SELECT
  COUNT(*) AS total,
  SUM(CASE WHEN json_extract(data, '$.success') = 0 THEN 1 ELSE 0 END) AS failures
FROM (
  SELECT data FROM events
  WHERE type = 'subagent_stop'
    AND json_extract(data, '$.agentType') LIKE ?
  ORDER BY ts DESC
  LIMIT ?
)
```

---

## 요구사항

### REQ-RA-201: SubagentStop 이벤트 기록

시스템은 SubagentStop 훅 이벤트를 수신하여 `events` 테이블에 기록해야 한다(SHALL).

기록 필드:
1. `type: "subagent_stop"` — 이벤트 타입 (SHALL)
2. `ts` — ISO 8601 타임스탬프 (SHALL)
3. `session_id` — stdin의 `session_id` (SHALL)
4. `project` — `getProjectName(input.cwd)` (SHALL)
5. `project_path` — stdin의 `cwd` (SHALL)
6. `data` JSON 내 `agentId` — stdin의 `agent_id` (SHALL)
7. `data` JSON 내 `agentType` — stdin의 `agent_type` (SHALL)
8. `data` JSON 내 `success` — 성공/실패 여부 (SHALL)

`db.mjs`의 `insertEvent()` 함수를 사용하여 기록한다(SHALL).

#### Scenario RA-201-1: 정상적인 SubagentStop 이벤트 기록

- **GIVEN** SubagentStop 이벤트가 stdin으로 `{ "session_id": "s1", "cwd": "/project", "agent_id": "a1", "agent_type": "executor" }` 제공
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** `events` 테이블에 `type: "subagent_stop"` 행이 INSERT된다

#### Scenario RA-201-2: stdin 필드 일부 누락 시

- **GIVEN** stdin에 `agent_type` 필드가 누락된 입력
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** 가능한 필드만으로 기록하고 정상 종료한다 (exit 0)

---

### REQ-RA-202: 서브에이전트 성능 통계 — SQL 집계

시스템은 별도의 통계 파일 없이 `events` 테이블에 대한 SQL 집계 쿼리로 에이전트 타입별 성능 통계를 제공해야 한다(SHALL).

1. SubagentStop 이벤트의 `data` JSON에 성공 여부(`success`)를 포함하여 기록한다(SHALL)
2. 성공/실패 판정: stdin의 응답 데이터 기반 (success 필드 또는 에러 유무로 판정) (SHALL)
3. 통계 조회는 `events` 테이블에 대한 SQL 집계로 수행한다(SHALL):
   ```sql
   SELECT json_extract(data, '$.agentType') AS agent_type,
          COUNT(*) AS total,
          SUM(CASE WHEN json_extract(data, '$.success') = 0 THEN 1 ELSE 0 END) AS failures
   FROM events WHERE type = 'subagent_stop'
   GROUP BY agent_type
   ```
4. 별도의 `subagent-stats.jsonl` 파일은 사용하지 않는다(SHALL)

#### Scenario RA-202-1: 성공한 서브에이전트 이벤트 기록

- **GIVEN** SubagentStop 이벤트에서 에러 없이 완료된 에이전트 정보
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** `events` 테이블에 `data.success = true`로 기록된다

#### Scenario RA-202-2: 실패한 서브에이전트 이벤트 기록

- **GIVEN** SubagentStop 이벤트에서 에러가 발생한 에이전트 정보
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** `events` 테이블에 `data.success = false`로 기록된다

#### Scenario RA-202-3: SQL 집계로 실패율 조회

- **GIVEN** `events` 테이블에 `executor` 타입의 이벤트가 10건(성공 7, 실패 3) 존재
- **WHEN** SQL 집계 쿼리로 통계를 조회하면
- **THEN** `{ agent_type: "executor", total: 10, failures: 3 }` 결과를 반환한다

---

### REQ-RA-203: 비차단 실행 보장

훅 스크립트는 어떤 상황에서도 exit 0으로 종료해야 한다(SHALL).

1. 전체 로직을 try-catch로 감싸야 한다(SHALL)
2. 예외 발생 시 조용히 `process.exit(0)`으로 종료 (SHALL)
3. stdin 파싱 실패, DB 쓰기 실패 등 모든 에러를 흡수 (SHALL)

#### Scenario RA-203-1: stdin 파싱 실패 시

- **GIVEN** stdin으로 유효하지 않은 JSON이 입력됨
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** exit code 0으로 정상 종료한다

#### Scenario RA-203-2: DB 쓰기 실패 시

- **GIVEN** `self-gen.db` 파일에 쓰기 권한이 없는 환경
- **WHEN** subagent-tracker.mjs가 실행되면
- **THEN** exit code 0으로 정상 종료한다

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
- `db.mjs` 모듈의 `insertEvent()`, `readStdin()`, `getProjectName()` 함수를 활용 (SHALL)
- stdout 출력 없음 — 이 훅은 컨텍스트 주입을 하지 않고 데이터 수집만 수행 (SHALL)

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| SubagentStop | Claude Code에서 서브에이전트 실행이 종료될 때 발생하는 훅 이벤트 |
| 에이전트 타입 (agentType) | 서브에이전트의 역할 구분 (executor, architect, designer 등) |
| SQL 집계 | events 테이블에 대한 GROUP BY 쿼리로 실시간 통계를 산출하는 방식 |
| 비차단 (Non-blocking) | 에러 발생 시에도 exit 0으로 종료하여 Claude Code 워크플로우를 방해하지 않는 특성 |
