---
id: batch-embeddings
title: "배치 임베딩 프로세서"
status: draft
created: 2026-02-09
domain: realtime-assist
depends: "realtime-assist/embedding-daemon, realtime-assist/error-kb, data-collection/log-writer"
constitution_version: "2.0.0"
---

# batch-embeddings

> `lib/batch-embeddings.mjs` — SessionEnd에서 detached `child_process.spawn`으로 실행되는 배치 임베딩 처리 스크립트. 에러 KB의 미임베딩 엔트리와 스킬 파일에 대해 벡터 임베딩을 일괄 생성하여 `vec_error_kb` 및 `vec_skill_embeddings` 가상 테이블에 저장한다. 실시간 훅의 2초 타임아웃 제약 없이 비동기 백그라운드에서 동작한다.

---

## 개요

배치 임베딩 프로세서는 실시간 훅에서 수행할 수 없는 벡터 임베딩 생성을 세션 종료 후 비동기로 처리한다. `session-summary.mjs`(REQ-DC-405)가 `spawn('node', [batchScript, projectPath], { detached: true, stdio: 'ignore' })`로 이 스크립트를 실행하면, 10초 지연 후 임베딩 데몬과 통신하여 에러 KB와 스킬 임베딩을 갱신한다.

### 파일 위치

- 스크립트: `~/.reflexion/lib/batch-embeddings.mjs`
- 데이터: `~/.reflexion/data/reflexion.db`

### 의존 모듈

| import | 출처 | 용도 |
|--------|------|------|
| `getDb`, `generateEmbeddings` | `./db.mjs` | DB 접근, 임베딩 생성 |
| `loadSkills`, `extractPatterns` | `./skill-matcher.mjs` | 스킬 파일 로드, 패턴 추출 |
| `isServerRunning`, `startServer` | `./embedding-client.mjs` | 임베딩 데몬 상태 확인 및 시작 |

### 실행 인자

| 인자 | 위치 | 설명 |
|------|------|------|
| `projectPath` | `process.argv[2]` | 프로젝트 경로. 미지정 시 `process.cwd()` 사용 |

### 관련 테이블

| 테이블 | 유형 | 용도 |
|--------|------|------|
| `error_kb` | 일반 | 에러 KB 원본 데이터 조회 |
| `vec_error_kb` | 가상 (sqlite-vec) | 에러 KB 벡터 임베딩 저장 |
| `skill_embeddings` | 일반 | 스킬 메타데이터 저장 |
| `vec_skill_embeddings` | 가상 (sqlite-vec) | 스킬 벡터 임베딩 저장 |

---

## 요구사항

### REQ-RA-601: 시작 지연 전략

시스템은 프로세스 시작 후 10초 지연(`setTimeout`)을 수행해야 한다(SHALL). 이는 `session-summary.mjs` 및 `ai-analyzer.mjs`의 동시 DB 쓰기 빈도를 줄이기 위한 것이다. 단, 이 지연만으로 경합을 완전히 방지하지는 않으며, 최종적으로는 SQLite WAL 모드와 `busy_timeout`(REQ-RA-602)에 의존한다.

#### Scenario: 10초 시작 지연

- **GIVEN** `session-summary.mjs`가 배치 임베딩 프로세스를 detached로 spawn한 상태
- **WHEN** `batch-embeddings.mjs` 프로세스가 시작되면
- **THEN** `await new Promise(r => setTimeout(r, 10000))`으로 10초 대기 후 본격 처리를 시작해야 한다(SHALL)

#### Scenario: 지연 중 프로세스 안정성

- **GIVEN** 10초 지연 중 시스템에 다른 이벤트가 발생하는 상태
- **WHEN** 지연 타이머가 만료되면
- **THEN** 정상적으로 다음 단계(임베딩 데몬 확인)로 진행해야 한다(SHALL)

---

### REQ-RA-602: DB busy_timeout 확장

시스템은 DB 연결 후 `busy_timeout`을 10,000ms로 설정해야 한다(SHALL). 이는 `session-summary` 및 AI 분석 프로세스와의 동시 쓰기 경합을 SQLite WAL 모드와 함께 처리하기 위한 것이다.

#### Scenario: busy_timeout 설정

- **GIVEN** 10초 시작 지연이 완료된 상태
- **WHEN** `getDb()`로 DB 연결을 획득하면
- **THEN** `db.pragma('busy_timeout = 10000')`을 실행하여 타임아웃을 10초로 확장해야 한다(SHALL)

#### Scenario: DB 쓰기 경합 발생

- **GIVEN** 다른 프로세스가 DB에 쓰기 중인 상태
- **WHEN** 배치 임베딩이 INSERT를 시도하면
- **THEN** SQLite WAL 모드 + busy_timeout 10초 범위 내에서 재시도하여 쓰기를 완료해야 한다(SHALL)

---

### REQ-RA-603: 임베딩 데몬 대기 루프

시스템은 임베딩 데몬이 실행 중인지 확인하고, 미실행 시 시작한 후 최대 15초간 대기해야 한다(SHALL).

대기 루프 절차:
1. `isServerRunning()`으로 데몬 상태를 확인한다 (SHALL)
2. 미실행 시 `startServer()`를 호출하여 데몬을 시작한다 (SHALL)
3. 최대 15회(1초 간격) 반복하여 `isServerRunning()`이 `true`를 반환할 때까지 대기한다 (SHALL)
4. 15초 후에도 데몬이 시작되지 않으면 이후 `generateEmbeddings()` 호출이 빈 배열을 반환하므로 임베딩 없이 종료된다

#### Scenario: 데몬이 이미 실행 중

- **GIVEN** 임베딩 데몬이 이미 실행 중인 상태
- **WHEN** `isServerRunning()`을 호출하면
- **THEN** `true`를 반환하고 대기 루프를 건너뛰어야 한다(SHALL)

#### Scenario: 데몬 시작 후 정상 대기

- **GIVEN** 임베딩 데몬이 실행 중이 아닌 상태
- **WHEN** `startServer()`를 호출하고 5초 후 데몬이 준비되면
- **THEN** 5번째 반복에서 `isServerRunning()`이 `true`를 반환하고 대기 루프를 종료해야 한다(SHALL)

#### Scenario: 데몬 시작 실패 (15초 타임아웃)

- **GIVEN** 임베딩 데몬이 시작에 실패하는 상태
- **WHEN** 15회 반복 후에도 `isServerRunning()`이 `false`를 반환하면
- **THEN** 대기 루프를 종료하고 다음 단계로 진행한다(SHALL). `generateEmbeddings()`는 빈 배열을 반환하므로 임베딩이 생성되지 않는다

---

### REQ-RA-604: 에러 KB 배치 임베딩

시스템은 `vec_error_kb`에 임베딩이 없는 `error_kb` 엔트리를 조회하여 벡터 임베딩을 배치 생성해야 한다(SHALL).

처리 절차:
1. 미임베딩 엔트리 조회 (SHALL):
   ```sql
   SELECT id, error_normalized FROM error_kb
   WHERE id NOT IN (SELECT error_kb_id FROM vec_error_kb)
   ```
2. `error_normalized` 텍스트 배열을 `generateEmbeddings(texts)`에 전달하여 벡터를 생성한다 (SHALL)
3. 각 엔트리에 대해 기존 벡터를 삭제 후 새 벡터를 삽입한다 (SHALL):
   ```sql
   DELETE FROM vec_error_kb WHERE error_kb_id = ?;
   INSERT INTO vec_error_kb (error_kb_id, embedding) VALUES (?, ?);
   ```
4. 임베딩 값은 `Buffer.from(new Float32Array(embedding).buffer)`로 변환한다 (SHALL)
5. 임베딩이 `null`/`undefined`인 엔트리는 건너뛴다 (SHALL)

#### Scenario: 미임베딩 에러 KB 엔트리 배치 처리

- **GIVEN** `error_kb`에 5건의 엔트리가 있고, 그 중 3건이 `vec_error_kb`에 임베딩이 없는 상태
- **WHEN** 에러 KB 배치 임베딩이 실행되면
- **THEN** 3건에 대해 `generateEmbeddings()`를 호출하고, 생성된 float[384] 벡터를 `vec_error_kb`에 저장해야 한다(SHALL)

#### Scenario: 미임베딩 엔트리가 없는 경우

- **GIVEN** `error_kb`의 모든 엔트리에 대응하는 `vec_error_kb` 임베딩이 이미 존재하는 상태
- **WHEN** 미임베딩 조회 쿼리를 실행하면
- **THEN** 빈 배열이 반환되고 임베딩 생성을 건너뛴다(SHALL)

#### Scenario: 부분 임베딩 생성 실패

- **GIVEN** 3건의 미임베딩 엔트리 중 2번째의 임베딩이 `null`로 반환되는 상태
- **WHEN** 배치 임베딩을 처리하면
- **THEN** 1번째와 3번째만 `vec_error_kb`에 저장하고, 2번째는 건너뛴다(SHALL)

---

### REQ-RA-605: 스킬 임베딩 갱신

시스템은 `loadSkills(projectPath)`로 전역 및 프로젝트 스킬을 로드하고, 각 스킬의 벡터 임베딩을 갱신해야 한다(SHALL).

처리 절차:
1. `loadSkills(projectPath)`로 스킬 목록을 로드한다 (SHALL)
2. 각 스킬에 대해 `skill_embeddings` 테이블에 메타데이터를 UPSERT한다 (SHALL):
   ```sql
   INSERT OR REPLACE INTO skill_embeddings (name, source_path, description, keywords, updated_at)
   VALUES (?, ?, ?, ?, ?)
   ```
   - `keywords`는 `JSON.stringify(extractPatterns(skill.content))`로 생성 (SHALL)
   - `updated_at`은 `new Date().toISOString()` (SHALL)
3. UPSERT 결과에서 `lastInsertRowid`를 획득하고, 없으면 `SELECT id FROM skill_embeddings WHERE name = ?`로 조회한다 (SHALL)
4. 스킬 콘텐츠의 처음 500자(`skill.content.slice(0, 500)`)로 임베딩을 생성한다 (SHALL)
5. 기존 벡터를 삭제 후 새 벡터를 삽입한다 (SHALL):
   ```sql
   DELETE FROM vec_skill_embeddings WHERE skill_id = ?;
   INSERT INTO vec_skill_embeddings (skill_id, embedding) VALUES (?, ?);
   ```
6. 임베딩 값은 `Buffer.from(new Float32Array(embedding).buffer)`로 변환한다 (SHALL)

#### Scenario: 스킬 임베딩 정상 갱신

- **GIVEN** `loadSkills(projectPath)`가 전역 스킬 2개, 프로젝트 스킬 1개를 반환하는 상태
- **WHEN** 스킬 임베딩 갱신이 실행되면
- **THEN** 3개 스킬 모두 `skill_embeddings` 메타데이터가 UPSERT되고, `vec_skill_embeddings`에 float[384] 벡터가 저장된다(SHALL)

#### Scenario: 기존 스킬 메타데이터 갱신 (UPSERT)

- **GIVEN** `skill_embeddings`에 `name = 'ts-init'` 엔트리가 이미 존재하는 상태
- **WHEN** 동일 스킬의 임베딩 갱신이 실행되면
- **THEN** `INSERT OR REPLACE`로 기존 행이 갱신되고, `vec_skill_embeddings`의 벡터도 교체된다(SHALL)

#### Scenario: 스킬 임베딩 생성 실패

- **GIVEN** `generateEmbeddings([text])`가 `null` 또는 빈 배열을 반환하는 상태
- **WHEN** 스킬 임베딩 갱신이 실행되면
- **THEN** 해당 스킬의 `skill_embeddings` 메타데이터는 저장되지만, `vec_skill_embeddings`에는 벡터가 저장되지 않는다(SHALL)

#### Scenario: skillId 획득 — lastInsertRowid 폴백

- **GIVEN** `INSERT OR REPLACE` 실행 후 `lastInsertRowid`가 0 또는 undefined인 상태
- **WHEN** skillId를 획득하려 하면
- **THEN** `SELECT id FROM skill_embeddings WHERE name = ?`로 조회하여 skillId를 획득해야 한다(SHALL)

#### Scenario: 스킬이 없는 경우

- **GIVEN** `loadSkills(projectPath)`가 빈 배열을 반환하는 상태
- **WHEN** 스킬 임베딩 갱신이 실행되면
- **THEN** 루프를 실행하지 않고 종료한다(SHALL)

---

### REQ-RA-606: Non-blocking 종료 보장

시스템은 모든 처리 완료 또는 예외 발생 시 `process.exit(0)`으로 종료해야 한다(SHALL). 배치 임베딩은 비핵심(non-critical) 프로세스이므로 어떤 오류도 exit code 0으로 종료한다.

#### Scenario: 정상 처리 완료

- **GIVEN** 에러 KB 및 스킬 임베딩이 모두 정상 처리된 상태
- **WHEN** 모든 처리가 완료되면
- **THEN** `process.exit(0)`으로 종료해야 한다(SHALL)

#### Scenario: 예외 발생 시 안전 종료

- **GIVEN** DB 접근, 임베딩 생성, 또는 기타 오류가 발생하는 상태
- **WHEN** 최상위 try-catch에서 예외가 포착되면
- **THEN** `process.exit(0)`으로 종료해야 한다(SHALL). 에러 로깅이나 재시도는 수행하지 않는다

---

## 비기능 요구사항

### 성능

- 전체 배치 처리는 60초 이내에 완료되어야 한다(SHOULD)
- 임베딩 데몬과의 통신은 건당 ~5ms를 목표로 한다(SHOULD)
- 에러 KB와 스킬 임베딩 처리는 순차적으로 실행한다(SHALL) — 에러 KB 먼저, 스킬 다음

### 안정성

- 전체 스크립트를 try-catch로 감싸 어떤 예외도 exit 0으로 처리한다(SHALL)
- 개별 임베딩 실패는 해당 엔트리를 건너뛰고 다음을 계속 처리한다(SHALL)
- DB 접근 실패 시 전체 프로세스를 조용히 종료한다(SHALL)

### 동시성

- WAL 모드 + `busy_timeout = 10000`으로 동시 쓰기 경합을 처리한다(SHALL)
- 10초 시작 지연으로 경합 빈도를 줄인다(SHALL). 단, 지연은 경합 방지를 보장하지 않는다

---

## 제약사항

- Detached 프로세스로만 실행된다 — 직접 import/호출 불가 (SHALL)
- `better-sqlite3` + `sqlite-vec` 사용 (SHALL)
- ES Modules 형식 (`.mjs` 확장자) (SHALL)
- 임베딩 차원: 384 (float 배열) (SHALL)
- 임베딩 생성에 `db.mjs`의 `generateEmbeddings()` 사용 (SHALL) — 내부적으로 `embedding-client.mjs`의 `embedViaServer()`를 호출
- 스킬 로드에 `skill-matcher.mjs`의 `loadSkills()`, `extractPatterns()` 사용 (SHALL)
- DB 경로: `~/.reflexion/data/reflexion.db` (SHALL)

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| 배치 임베딩 | SessionEnd 이후 비동기로 실행되는 벡터 임베딩 일괄 생성 프로세스 |
| Detached 프로세스 | `child_process.spawn`의 `detached: true` 옵션으로 부모 프로세스와 분리된 자식 프로세스 |
| 임베딩 데몬 | Transformers.js 모델을 상주 로드하여 Unix socket으로 임베딩을 제공하는 서버 |
| `vec_error_kb` | sqlite-vec 가상 테이블로 에러 KB의 384차원 벡터 임베딩을 저장 |
| `vec_skill_embeddings` | sqlite-vec 가상 테이블로 스킬의 384차원 벡터 임베딩을 저장 |
| busy_timeout | SQLite의 잠금 대기 시간 설정. 다른 프로세스가 DB를 잠근 경우 지정 시간까지 재시도 |
| WAL 모드 | Write-Ahead Logging. 읽기/쓰기 동시 접근을 허용하는 SQLite 저널 모드 |
| UPSERT | INSERT 시 충돌이 발생하면 UPDATE로 전환하는 SQL 패턴 (`INSERT OR REPLACE`) |
