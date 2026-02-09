---
id: log-writer
title: "log-writer"
status: draft
created: 2026-02-07
domain: data-collection
depends: "better-sqlite3, sqlite-vec, @xenova/transformers"
constitution_version: "2.0.0"
---

# log-writer

> 데이터베이스 관리 모듈 (`lib/db.mjs`). 모든 데이터 수집 훅의 기반이 되는 공통 라이브러리로, SQLite DB 연결 관리, 이벤트 삽입/조회/삭제, 설정 로딩, stdin JSON 파싱, 프라이버시 태그 스트리핑, 임베딩 생성 및 벡터 검색 기능을 제공한다.

---

## Requirement: REQ-DC-001 — 이벤트 삽입 (insertEvent)

시스템은 이벤트 데이터를 `events` 테이블에 INSERT(SHALL)해야 한다. 각 이벤트는 공통 필드(`v`, `type`, `ts`, `session_id`, `project`, `project_path`)와 타입별 데이터(`data` JSON 컬럼)로 구성된다. `events` 테이블은 INSERT-only이며, UPDATE는 DB 트리거로 금지(SHALL)된다.

### Scenario: 정상 이벤트 삽입

- **GIVEN** `~/.self-generation/data/self-gen.db` 데이터베이스가 초기화된 상태
- **WHEN** `insertEvent(entry)`를 호출하면
- **THEN** `events` 테이블에 새 행이 INSERT(SHALL)된다. `entry`의 공통 필드(`v`, `type`, `ts`, `sessionId`, `project`, `projectPath`)는 각 컬럼에, 나머지 타입별 필드(`...rest`)는 `data` JSON 컬럼에 `JSON.stringify(rest)`로 저장된다

### Scenario: 데이터베이스 자동 초기화

- **GIVEN** `~/.self-generation/data/self-gen.db` 파일이 존재하지 않는 상태
- **WHEN** `insertEvent()`가 호출되면
- **THEN** `getDb()`를 통해 DB가 자동 생성 및 초기화(SHALL)된 후 이벤트가 삽입된다

### Scenario: 스키마 버전 포함

- **GIVEN** 어떤 훅에서든 이벤트를 생성할 때
- **WHEN** `insertEvent()`로 기록하면
- **THEN** 모든 이벤트에 `v: 1` 값이 포함(SHALL)되어야 한다

---

## Requirement: REQ-DC-002 — 이벤트 조회 (queryEvents)

시스템은 `events` 테이블에서 다양한 필터 조건으로 이벤트를 조회(SHALL)할 수 있어야 한다. 필터 조건: `type`, `sessionId`, `project`, `projectPath`, `since` (타임스탬프), `search` (FTS5 전문 검색), `limit`.

### Scenario: 필터 조건으로 이벤트 조회

- **GIVEN** `events` 테이블에 여러 세션, 프로젝트의 이벤트가 혼재된 상태
- **WHEN** `queryEvents({ type: 'prompt', sessionId: 'abc-123' })`을 호출하면
- **THEN** `WHERE type = ? AND session_id = ?` 파라미터화된 쿼리로 일치하는 이벤트만 반환(SHALL)된다

### Scenario: 최근 N개 제한 조회

- **GIVEN** `events` 테이블에 1000개 이상의 이벤트가 있는 상태
- **WHEN** `queryEvents({ limit: 100 })`을 호출하면
- **THEN** `ORDER BY ts DESC LIMIT 100`으로 최근 100개 이벤트만 반환(SHALL)된다

### Scenario: since 필터 조회

- **GIVEN** `events` 테이블에 다양한 시점의 이벤트가 있는 상태
- **WHEN** `since` 필터가 지정되면
- **THEN** `WHERE ts >= ?` 조건으로 해당 시점 이후의 이벤트만 반환(SHALL)된다

### Scenario: FTS5 전문 검색 (v9)

- **GIVEN** `events_fts` 가상 테이블이 `events` 테이블과 동기화된 상태
- **WHEN** `queryEvents({ search: '키워드' })`를 호출하면
- **THEN** `id IN (SELECT rowid FROM events_fts WHERE events_fts MATCH ?)` 서브쿼리로 전문 검색 결과를 반환(SHALL)한다

### Scenario: 결과 포맷 역호환

- **GIVEN** `queryEvents()`가 결과를 반환할 때
- **WHEN** 행을 변환하면
- **THEN** `row.data`를 `JSON.parse()`하여 공통 필드와 병합한 플랫 객체를 반환(SHALL)한다 (`{ v, type, ts, sessionId, project, projectPath, ...data }`)

---

## Requirement: REQ-DC-003 — DB 초기화 (initDb)

시스템은 SQLite 데이터베이스의 테이블, 인덱스, FTS5 가상 테이블, vec0 가상 테이블을 초기화(SHALL)해야 한다.

### Scenario: 최초 DB 생성 시 스키마 초기화

- **GIVEN** `self-gen.db` 파일이 존재하지 않거나 `events` 테이블이 없는 상태
- **WHEN** `initDb(db)`가 호출되면
- **THEN** 다음 테이블과 인덱스가 생성(SHALL)된다:
  ```sql
  -- events 테이블 + 4개 인덱스
  CREATE TABLE IF NOT EXISTS events (...);
  CREATE INDEX IF NOT EXISTS idx_events_session ON events(session_id, ts);
  CREATE INDEX IF NOT EXISTS idx_events_project_type ON events(project_path, type, ts);
  CREATE INDEX IF NOT EXISTS idx_events_type_ts ON events(type, ts);
  CREATE INDEX IF NOT EXISTS idx_events_session_type ON events(session_id, type);

  -- error_kb 테이블 + UNIQUE 인덱스
  CREATE TABLE IF NOT EXISTS error_kb (...);
  CREATE UNIQUE INDEX IF NOT EXISTS idx_error_kb_error ON error_kb(error_normalized);

  -- feedback 테이블
  CREATE TABLE IF NOT EXISTS feedback (...);

  -- analysis_cache 테이블 + content-addressable UNIQUE 인덱스
  CREATE TABLE IF NOT EXISTS analysis_cache (...);
  CREATE UNIQUE INDEX IF NOT EXISTS idx_analysis_cache_hash ON analysis_cache(project, days, input_hash);

  -- skill_embeddings 테이블
  CREATE TABLE IF NOT EXISTS skill_embeddings (...);
  ```

### Scenario: FTS5 전문 검색 인덱스 생성 (v9)

- **GIVEN** `initDb(db)`가 호출되는 상태
- **WHEN** 스키마 초기화가 실행되면
- **THEN** `events_fts` FTS5 가상 테이블과 INSERT/DELETE 동기화 트리거, UPDATE 금지 트리거가 생성(SHALL)된다

### Scenario: vec0 가상 테이블 생성

- **GIVEN** `initDb(db)`가 호출되는 상태
- **WHEN** 스키마 초기화가 실행되면
- **THEN** `vec_error_kb`와 `vec_skill_embeddings` vec0 가상 테이블이 생성(SHALL)된다. 생성 실패 시(이미 존재 또는 vec0 미지원) try-catch로 무시한다

### Scenario: WAL 모드 활성화

- **GIVEN** DB 연결이 생성된 상태
- **WHEN** `getDb()` 내부에서 초기화가 수행되면
- **THEN** `PRAGMA journal_mode=WAL`과 `PRAGMA busy_timeout=5000`이 설정(SHALL)된다

---

## Requirement: REQ-DC-004 — 보관 기한 초과 이벤트 삭제 (pruneOldEvents)

시스템은 보관 기한이 경과한 이벤트와 미사용 에러 KB 항목을 SQL DELETE로 삭제(SHALL)해야 한다.

### Scenario: 보관 기한 초과 이벤트 정리

- **GIVEN** `events` 테이블에 90일 이전의 이벤트가 존재하는 상태
- **WHEN** `pruneOldEvents(retentionDays)`가 호출되면
- **THEN** `DELETE FROM events WHERE ts < ?` SQL이 실행(SHALL)되어 보관 기한 초과 이벤트가 삭제된다

### Scenario: 미사용 에러 KB 항목 정리

- **GIVEN** `error_kb` 테이블에 보관 기한 초과이며 `use_count = 0`인 항목이 존재하는 상태
- **WHEN** `pruneOldEvents(retentionDays)`가 호출되면
- **THEN** `DELETE FROM error_kb WHERE ts < ? AND use_count = 0` SQL이 실행(SHALL)되어 미사용 항목이 삭제된다

### Scenario: 설정 기반 보관 기한

- **GIVEN** `retentionDays` 파라미터가 `undefined`인 상태
- **WHEN** `pruneOldEvents()`가 호출되면
- **THEN** `loadConfig().retentionDays` 또는 기본값 90일을 사용(SHALL)한다

---

## Requirement: REQ-DC-005 — 설정 로딩 (loadConfig, isEnabled)

시스템은 `~/.self-generation/config.json` 파일에서 설정을 로드하는 `loadConfig()` 유틸리티 함수와, 시스템 활성화 여부를 확인하는 `isEnabled()` 함수를 제공(SHALL)해야 한다.

### 설정 필드

- `enabled` (boolean, 기본값: true): 시스템 전체 활성화 여부
- `collectPromptText` (boolean, 기본값: true): 프롬프트 원문 수집 여부
- `retentionDays` (number, 기본값: 90): 이벤트 보관 기한 (일)
- `dbPath` (string, 기본값: `~/.self-generation/data/self-gen.db`): SQLite DB 파일 경로
- `analysisOnSessionEnd` (boolean, 기본값: true): 세션 종료 시 AI 분석 자동 실행 여부
- `analysisDays` (number, 기본값: 7): AI 분석 대상 기간 (일)
- `analysisCacheMaxAgeHours` (number, 기본값: 24): AI 분석 캐시 유효 기간 (시간)
- `embedding` (object, 기본값: `{ model: 'claude', dimensions: 384 }`): 임베딩 생성 설정

### Scenario: config.json 미존재 시 기본값 사용

- **GIVEN** `~/.self-generation/config.json` 파일이 존재하지 않는 상태
- **WHEN** `loadConfig()`를 호출하면
- **THEN** 빈 객체 `{}`를 반환(SHALL)한다 (각 사용처에서 기본값 적용)

### Scenario: isEnabled() — 활성화 상태 확인

- **GIVEN** `config.json`에서 `enabled: false`로 설정된 상태
- **WHEN** `isEnabled()`를 호출하면
- **THEN** `false`를 반환(SHALL)하고, 각 훅은 즉시 `process.exit(0)`으로 종료한다

### Scenario: isEnabled() — 기본 활성화

- **GIVEN** `config.json`이 없거나 `enabled` 필드가 명시되지 않은 상태
- **WHEN** `isEnabled()`를 호출하면
- **THEN** `true`를 반환(SHALL)한다

---

## Requirement: REQ-DC-006 — stdin JSON 파싱 (readStdin)

시스템은 Claude Code Hook이 전달하는 stdin JSON 데이터를 비동기로 읽고 파싱(SHALL)해야 한다. Promise 기반으로 동작하며, 5초 타임아웃을 포함한다.

### Scenario: 정상 stdin 읽기

- **GIVEN** Claude Code Hook이 JSON 데이터를 stdin으로 전달하는 상태
- **WHEN** `await readStdin()`을 호출하면
- **THEN** `process.stdin`에서 데이터를 수집하고, `end` 이벤트 시 `JSON.parse()`로 파싱하여 JavaScript 객체를 반환(SHALL)한다

### Scenario: stdin 타임아웃

- **GIVEN** stdin 데이터가 5초 이내에 도착하지 않는 상태
- **WHEN** 타임아웃이 발생하면
- **THEN** `reject(new Error('stdin timeout'))`로 에러를 반환(SHALL)한다

---

## Requirement: REQ-DC-007 — 싱글톤 DB 연결 (getDb)

시스템은 프로세스 내에서 단일 SQLite DB 연결을 싱글톤으로 관리(SHALL)해야 한다. 테이블이 이미 존재하면 DDL 생성을 건너뛰어 5~10ms를 절약한다.

### Scenario: 최초 호출 시 DB 연결 생성

- **GIVEN** 프로세스에서 `getDb()`가 아직 호출되지 않은 상태
- **WHEN** `getDb()`를 호출하면
- **THEN** `better-sqlite3`를 사용하여 DB 연결을 생성하고, WAL 모드와 `busy_timeout`을 설정하고, `sqlite-vec`를 로드하고, 테이블 존재 여부를 확인하여 필요 시 `initDb()`를 호출(SHALL)한 후 연결 객체를 반환한다

### Scenario: 재호출 시 기존 연결 반환

- **GIVEN** `getDb()`가 이미 호출되어 연결이 존재하는 상태
- **WHEN** `getDb()`를 다시 호출하면
- **THEN** 새 연결을 생성하지 않고 기존 연결 객체를 반환(SHALL)한다

### Scenario: v8→v9 마이그레이션 자동 실행

- **GIVEN** DB 연결이 최초 생성되는 상태
- **WHEN** `getDb()` 내부에서 초기화가 완료되면
- **THEN** `migrateV9(db)`를 호출하여 `analysis_cache`에 `input_hash` 컬럼과 `events_fts` 가상 테이블이 없으면 추가(SHALL)한다

---

## Requirement: REQ-DC-008 — 임베딩 생성 (generateEmbeddings)

시스템은 텍스트 배열을 입력받아 임베딩 데몬(embedding-server.mjs)을 통해 벡터 임베딩을 생성(SHALL)해야 한다. 이 함수는 비동기(async)로 동작하며, 데몬 미실행 시 빈 배열을 반환하여 텍스트 매칭 폴백이 동작하도록 한다.

### Scenario: 텍스트 배열 임베딩 생성

- **GIVEN** 임베딩 데몬이 실행 중인 상태
- **WHEN** `await generateEmbeddings(['에러 메시지 A', '에러 메시지 B'])`를 호출하면
- **THEN** `embedding-client.mjs`의 `embedViaServer(texts)`를 사용하여 각 텍스트에 대해 float[384] 벡터 배열을 반환(SHALL)한다

### Scenario: 빈 배열 입력

- **GIVEN** 빈 배열 `[]`이 입력된 상태
- **WHEN** `await generateEmbeddings([])`를 호출하면
- **THEN** 빈 배열 `[]`을 즉시 반환(SHALL)한다

### Scenario: 데몬 미실행 시 폴백

- **GIVEN** 임베딩 데몬이 실행되지 않은 상태
- **WHEN** `await generateEmbeddings(texts)`를 호출하면
- **THEN** try-catch로 에러를 포착하고 빈 배열 `[]`을 반환(SHALL)하여 텍스트 매칭 폴백이 동작한다

---

## Requirement: REQ-DC-009 — 벡터 유사도 검색 (vectorSearch)

시스템은 VEC_TABLE_REGISTRY를 통해 확장 가능한 2단계 벡터 유사도 검색을 수행(SHALL)할 수 있어야 한다. 쿼리 임베딩은 `Buffer.from(new Float32Array(...).buffer)`로 변환하여 바인딩한다(SHALL).

### Scenario: 2단계 벡터 검색

- **GIVEN** vec0 가상 테이블에 임베딩이 저장된 상태
- **WHEN** `vectorSearch(table, vecTable, queryEmbedding, limit)`를 호출하면
- **THEN** Step 1에서 vec0 가상 테이블의 `MATCH` 연산자로 KNN 검색하여 ID + distance를 수집하고, Step 2에서 원본 테이블에서 ID로 상세 데이터를 조회하여 distance를 병합한 결과를 반환(SHALL)한다

### Scenario: VEC_TABLE_REGISTRY 조회

- **GIVEN** `table` 파라미터가 `'error_kb'`인 상태
- **WHEN** `vectorSearch('error_kb', ...)`를 호출하면
- **THEN** 레지스트리에서 `{ vecTable: 'vec_error_kb', fkColumn: 'error_kb_id' }`를 조회하여 동적으로 쿼리를 구성(SHALL)한다

### Scenario: 미등록 테이블

- **GIVEN** `VEC_TABLE_REGISTRY`에 등록되지 않은 테이블이 요청된 상태
- **WHEN** `vectorSearch('unknown_table', ...)`를 호출하면
- **THEN** 빈 배열 `[]`을 즉시 반환(SHALL)한다

### Scenario: 결과 없음

- **GIVEN** vec0 가상 테이블이 비어있는 상태
- **WHEN** `vectorSearch(table, vecTable, queryEmbedding, limit)`를 호출하면
- **THEN** 빈 배열 `[]`을 반환(SHALL)한다

---

## Requirement: REQ-DC-010 — 세션 이벤트 조회 (getSessionEvents)

시스템은 특정 세션의 이벤트를 효율적으로 조회(SHALL)할 수 있어야 한다. 내부적으로 `queryEvents()`를 위임 호출한다.

### Scenario: 세션 ID로 이벤트 조회

- **GIVEN** `events` 테이블에 여러 세션의 이벤트가 있는 상태
- **WHEN** `getSessionEvents(sessionId, limit)`를 호출하면
- **THEN** `queryEvents({ sessionId, limit })`를 호출하여 해당 세션의 최근 이벤트만 반환(SHALL)된다

---

## Requirement: REQ-DC-011 — 프라이버시 태그 스트리핑 (stripPrivateTags)

시스템은 `<private>...</private>` 태그로 감싼 텍스트를 `[PRIVATE]`로 치환하여 민감 정보가 DB에 저장되지 않도록 보장(SHALL)해야 한다.

### Scenario: private 태그 스트리핑

- **GIVEN** 프롬프트 텍스트에 `"비밀번호는 <private>abc123</private>입니다"` 문자열이 포함된 상태
- **WHEN** `stripPrivateTags(text)`를 호출하면
- **THEN** `"비밀번호는 [PRIVATE]입니다"`를 반환(SHALL)한다

### Scenario: 닫히지 않은 태그 무시

- **GIVEN** 프롬프트 텍스트에 `"<private>비밀번호"` (닫는 태그 없음)가 포함된 상태
- **WHEN** `stripPrivateTags(text)`를 호출하면
- **THEN** 원본 텍스트를 그대로 반환(SHALL)한다 (명시적 범위 지정 시에만 스트리핑)

### Scenario: null/빈 입력

- **GIVEN** `null` 또는 `undefined`가 입력된 상태
- **WHEN** `stripPrivateTags(text)`를 호출하면
- **THEN** 입력값을 그대로 반환(SHALL)한다

---

## Requirement: REQ-DC-012 — 프로젝트 경로 유틸 (getProjectName, getProjectPath)

시스템은 프로젝트 식별을 위한 유틸리티 함수를 제공(SHALL)해야 한다.

### Scenario: getProjectName — 디렉토리명 추출

- **GIVEN** `cwd`가 `/Users/user/projects/my-app`인 상태
- **WHEN** `getProjectName(cwd)`를 호출하면
- **THEN** `'my-app'`을 반환(SHALL)한다 (표시용)

### Scenario: getProjectPath — 프로젝트 루트 경로

- **GIVEN** `CLAUDE_PROJECT_DIR` 환경변수가 설정된 상태
- **WHEN** `getProjectPath(cwd)`를 호출하면
- **THEN** `process.env.CLAUDE_PROJECT_DIR`을 우선 반환(SHALL)한다 (cwd가 서브디렉토리일 수 있으므로)

### Scenario: getProjectPath — 환경변수 미설정

- **GIVEN** `CLAUDE_PROJECT_DIR` 환경변수가 없는 상태
- **WHEN** `getProjectPath(cwd)`를 호출하면
- **THEN** `cwd`를 그대로 반환(SHALL)한다

---

## 비고

- 이 모듈은 모든 데이터 수집 훅(`prompt-logger`, `tool-logger`, `error-logger`, `session-summary`)의 공통 의존성이다
- `better-sqlite3`, `sqlite-vec`, `@xenova/transformers` 3개 외부 의존성을 사용한다
- `getProjectName(cwd)`는 표시용. 정규 식별에는 `getProjectPath(cwd)` 기반 `projectPath` 사용을 권장한다
- SQLite WAL 모드는 훅 프로세스 간 동시 쓰기 충돌을 방지한다
- JSONL 로테이션 관련 기능(`rotateIfNeeded`, `getLogFile`)은 SQLite 전환으로 제거되었다
- `VEC_TABLE_REGISTRY`를 통해 새로운 벡터 테이블 추가 시 하드코딩 없이 확장 가능하다
- `migrateV9()`는 v8→v9 마이그레이션으로 `input_hash` 컬럼과 `events_fts` 가상 테이블을 안전하게 추가한다 (멱등성 보장)
