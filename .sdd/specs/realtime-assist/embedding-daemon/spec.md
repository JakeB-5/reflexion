---
id: embedding-daemon
title: "Embedding Daemon"
status: draft
created: 2026-02-09
domain: realtime-assist
depends: "data-collection/log-writer"
constitution_version: "2.0.0"
---

# Embedding Daemon

> Transformers.js 모델을 메모리에 상주시키는 Unix socket 기반 임베딩 서버(`lib/embedding-server.mjs`)와, 훅 프로세스에서 서버와 통신하는 경량 클라이언트(`lib/embedding-client.mjs`)의 통합 스펙. 훅 프로세스는 매 호출마다 새로 생성되므로 모델 로딩(1~4초)을 매번 반복하는 대신, 상주 프로세스의 소켓 인터페이스를 통해 ~5ms로 임베딩을 생성한다.

---

## 서버 요구사항 (embedding-server.mjs)

### REQ-RA-501: Unix Socket 서버 생성

시스템은 Node.js `net.createServer()`를 사용하여 Unix domain socket 서버를 생성해야 한다(SHALL). 소켓 경로는 `/tmp/self-gen-embed.sock` 고정값을 사용한다(SHALL).

#### Scenario: 서버 정상 시작

- **GIVEN** 소켓 경로 `/tmp/self-gen-embed.sock`에 기존 소켓 파일이 없는 상태
- **WHEN** `embedding-server.mjs`가 실행된다
- **THEN** 모델 로딩 완료 후 `server.listen(SOCKET_PATH)`로 소켓 서버가 시작되고(SHALL), stderr에 `[embedding-server] Listening on /tmp/self-gen-embed.sock` 메시지가 출력된다

#### Scenario: stale 소켓 파일 정리

- **GIVEN** 이전 프로세스가 비정상 종료하여 `/tmp/self-gen-embed.sock` 파일이 남아 있는 상태
- **WHEN** `embedding-server.mjs`가 시작된다
- **THEN** `existsSync(SOCKET_PATH)` 확인 후 `unlinkSync(SOCKET_PATH)`로 stale 소켓 파일을 삭제한 후(SHALL) 서버를 시작한다

---

### REQ-RA-502: Transformers.js 모델 로딩

시스템은 `@xenova/transformers`의 `pipeline('feature-extraction', ...)` API로 `Xenova/paraphrase-multilingual-MiniLM-L12-v2` 모델을 로딩해야 한다(SHALL). 모델 캐시 디렉토리는 `~/.self-generation/models/`로 설정한다(SHALL). 모델은 384차원 벡터를 생성한다.

**실행 순서**: stale socket 정리(REQ-RA-502) → 모델 로딩(REQ-RA-501) → `server.listen()` 순서로 실행해야 한다(SHALL).

#### Scenario: 최초 모델 로딩

- **GIVEN** 서버 프로세스가 시작된 상태
- **WHEN** `init()` 함수가 호출된다
- **THEN** `env.cacheDir`을 `~/.self-generation/models/`로 설정하고(SHALL), `pipeline('feature-extraction', 'Xenova/paraphrase-multilingual-MiniLM-L12-v2')`로 모델을 로딩한다. stderr에 `[embedding-server] Loading model...` → `[embedding-server] Model loaded, ready for requests` 메시지를 순서대로 출력한다(SHALL)

#### Scenario: 텍스트 임베딩 생성

- **GIVEN** 모델이 로딩된 상태
- **WHEN** `embed(['텍스트 A', '텍스트 B'])`가 호출된다
- **THEN** 각 텍스트에 대해 `extractor(text, { pooling: 'mean', normalize: true })`를 호출하여 float[384] 벡터를 생성한다(SHALL). `output.data`에서 `Array.from()`으로 변환하고, `isFinite` 검증에 실패하면 `null`을 반환한다(SHALL)

#### Scenario: 빈 텍스트 처리

- **GIVEN** 모델이 로딩된 상태
- **WHEN** `embed(['', null, '  '])`처럼 빈/null/공백 텍스트가 포함된 배열이 입력된다
- **THEN** 해당 항목에 대해 임베딩 생성을 건너뛰고 `null`을 반환한다(SHALL)

---

### REQ-RA-503: Newline-Delimited JSON 프로토콜

시스템은 클라이언트와 newline-delimited JSON(`\n`) 프로토콜로 통신해야 한다(SHALL). 각 요청과 응답은 JSON 한 줄 + `\n`으로 구분된다.

#### Scenario: 임베딩 요청/응답

- **GIVEN** 클라이언트가 소켓에 연결된 상태
- **WHEN** `{"action":"embed","texts":["에러 메시지 A","에러 메시지 B"]}\n`을 전송한다
- **THEN** 서버는 `{"embeddings":[[0.1,...],[0.2,...]]}\n`을 응답하고 연결을 종료한다(SHALL)

#### Scenario: Health Check 요청/응답

- **GIVEN** 클라이언트가 소켓에 연결된 상태
- **WHEN** `{"action":"health"}\n`을 전송한다
- **THEN** 서버는 `{"status":"ok"}\n`을 응답하고 연결을 종료한다(SHALL)

#### Scenario: 알 수 없는 action

- **GIVEN** 클라이언트가 소켓에 연결된 상태
- **WHEN** `{"action":"unknown"}\n`을 전송한다
- **THEN** 서버는 `{"error":"unknown action"}\n`을 응답하고 연결을 종료한다(SHALL)

#### Scenario: 잘못된 JSON 입력

- **GIVEN** 클라이언트가 소켓에 연결된 상태
- **WHEN** 파싱 불가능한 데이터를 전송한다
- **THEN** 서버는 `{"error":"<에러 메시지>"}\n`을 응답한다(SHALL). 서버 프로세스는 종료되지 않는다(SHALL NOT)

---

### REQ-RA-504: Idle Timeout 자동 종료

시스템은 마지막 요청 이후 30분(1,800,000ms) 동안 새 요청이 없으면 자동으로 종료해야 한다(SHALL). 이를 통해 리소스를 불필요하게 점유하는 것을 방지한다.

#### Scenario: Idle Timeout 만료 시 자동 종료

- **GIVEN** 서버가 실행 중이고 마지막 요청으로부터 30분이 경과한 상태
- **WHEN** idle timer가 만료된다
- **THEN** stderr에 `[embedding-server] Idle timeout, shutting down`을 출력하고(SHALL), `server.close()` 후 `process.exit(0)`으로 정상 종료한다(SHALL)

#### Scenario: 요청 수신 시 타이머 리셋

- **GIVEN** 서버가 실행 중이고 idle timer가 진행 중인 상태
- **WHEN** 새 클라이언트 연결이 수립된다
- **THEN** 기존 타이머를 `clearTimeout()`으로 취소하고 새 30분 타이머를 시작한다(SHALL)

#### Scenario: 서버 시작 시 초기 타이머 설정

- **GIVEN** 서버가 `server.listen()` 콜백에 진입한 상태
- **WHEN** 서버가 리스닝을 시작한다
- **THEN** 최초 idle timer가 설정된다(SHALL)

---

### REQ-RA-505: EADDRINUSE 처리

시스템은 `server.on('error')` 이벤트에서 `EADDRINUSE` 에러를 감지하여 다른 서버 인스턴스가 이미 실행 중임을 판단하고, 중복 실행 없이 종료해야 한다(SHALL).

#### Scenario: 다른 인스턴스 실행 중

- **GIVEN** 다른 embedding-server 프로세스가 이미 소켓을 점유한 상태
- **WHEN** 새 embedding-server 프로세스가 `server.listen()`을 시도한다
- **THEN** `EADDRINUSE` 에러를 감지하고 stderr에 `[embedding-server] Socket already in use, another instance running. Exiting.`을 출력한 후(SHALL) `process.exit(0)`으로 종료한다(SHALL)

#### Scenario: 기타 서버 에러

- **GIVEN** 서버에서 `EADDRINUSE`가 아닌 다른 에러가 발생한 상태
- **WHEN** `server.on('error')` 핸들러가 실행된다
- **THEN** 해당 에러를 `throw`하여 프로세스를 비정상 종료한다(SHALL)

---

### REQ-RA-506: Graceful Shutdown

시스템은 `SIGTERM` 및 `SIGINT` 시그널을 수신하면 소켓 서버를 정리하고 정상 종료해야 한다(SHALL).

#### Scenario: SIGTERM 수신 시 종료

- **GIVEN** 서버가 실행 중인 상태
- **WHEN** `SIGTERM` 시그널을 수신한다
- **THEN** `server.close()`를 호출하여 새 연결 수락을 중단하고 `process.exit(0)`으로 종료한다(SHALL)

#### Scenario: SIGINT 수신 시 종료

- **GIVEN** 서버가 실행 중인 상태
- **WHEN** `SIGINT` 시그널을 수신한다
- **THEN** `server.close()`를 호출하여 새 연결 수락을 중단하고 `process.exit(0)`으로 종료한다(SHALL)

#### Scenario: 클라이언트 연결 에러 무시

- **GIVEN** 클라이언트가 서버에 연결된 상태
- **WHEN** 클라이언트가 비정상 종료(disconnect)한다
- **THEN** `conn.on('error', () => {})` 핸들러로 에러를 무시하고(SHALL), 서버는 계속 실행된다

---

## 클라이언트 요구사항 (embedding-client.mjs)

### REQ-RA-507: embedViaServer 인터페이스

시스템은 `embedViaServer(texts)` 함수를 export하여 텍스트 배열을 입력받고 임베딩 벡터 배열을 반환해야 한다(SHALL). 이 함수는 비동기(async)로 동작한다.

#### Scenario: 정상 임베딩 요청

- **GIVEN** 임베딩 서버가 실행 중인 상태
- **WHEN** `await embedViaServer(['텍스트 A', '텍스트 B'])`를 호출한다
- **THEN** 소켓으로 `{"action":"embed","texts":["텍스트 A","텍스트 B"]}\n`을 전송하고(SHALL), 서버 응답에서 `embeddings` 필드를 추출하여 `number[][]` 배열을 반환한다(SHALL)

#### Scenario: 서버 미실행 시 자동 시작 후 재시도

- **GIVEN** 임베딩 서버가 실행 중이 아닌 상태
- **WHEN** `await embedViaServer(texts)`를 호출한다
- **THEN** 소켓 연결에서 `ECONNREFUSED` 또는 `ENOENT` 에러가 발생하면(SHALL), `startServer()`로 데몬을 시작하고, 5초(`5000ms`) 대기 후 1회 재시도한다(SHALL)

#### Scenario: 재시도 후에도 실패

- **GIVEN** `startServer()` 이후에도 서버 연결이 실패하는 상태
- **WHEN** 재시도 요청이 실패한다
- **THEN** 에러를 throw한다(SHALL). 호출자(`generateEmbeddings`)에서 try-catch로 빈 배열 폴백을 처리한다

---

### REQ-RA-508: 소켓 통신 및 타임아웃

시스템은 `_sendRequest(texts)` 내부 함수로 소켓 통신을 수행해야 한다(SHALL). 10초(10,000ms) 타임아웃을 적용하여 무한 대기를 방지한다(SHALL).

#### Scenario: 정상 소켓 통신

- **GIVEN** 서버가 소켓에서 리스닝 중인 상태
- **WHEN** `_sendRequest(texts)`가 호출된다
- **THEN** `createConnection(SOCKET_PATH)`로 연결하고(SHALL), `{"action":"embed","texts":[...]}\n`을 전송하고, 응답 데이터를 수집하여 `end` 이벤트에서 JSON 파싱 후 `embeddings` 필드를 resolve한다(SHALL)

#### Scenario: 타임아웃 발생

- **GIVEN** 서버가 10초 내에 응답하지 않는 상태
- **WHEN** 타임아웃 타이머가 만료된다
- **THEN** `conn.destroy()`로 연결을 강제 종료하고 `'Embedding server timeout'` 에러로 reject한다(SHALL)

#### Scenario: 서버 응답에 embeddings 필드 없음

- **GIVEN** 서버가 에러 응답(`{"error":"..."}\n`)을 반환한 상태
- **WHEN** 응답 JSON에 `embeddings` 필드가 없다
- **THEN** `res.error` 또는 `'No embeddings'` 메시지로 reject한다(SHALL)

---

### REQ-RA-509: isServerRunning 상태 확인

시스템은 `isServerRunning()` 함수를 export하여 임베딩 데몬의 실행 상태를 확인해야 한다(SHALL). Health check 프로토콜을 사용한다.

#### Scenario: 서버 실행 중

- **GIVEN** 임베딩 서버가 소켓에서 리스닝 중인 상태
- **WHEN** `await isServerRunning()`을 호출한다
- **THEN** 소켓에 연결하여 `{"action":"health"}\n`을 전송하고(SHALL), 응답에서 `status === 'ok'`이면 `true`를 반환한다(SHALL)

#### Scenario: 서버 미실행

- **GIVEN** 임베딩 서버가 실행 중이 아닌 상태
- **WHEN** `await isServerRunning()`을 호출한다
- **THEN** 소켓 연결 에러가 발생하고 `false`를 반환한다(SHALL)

#### Scenario: Health Check 타임아웃

- **GIVEN** 소켓 연결은 됐으나 서버가 500ms 내에 응답하지 않는 상태
- **WHEN** health check 타임아웃이 만료된다
- **THEN** `conn.destroy()` 후 `false`를 반환한다(SHALL)

---

### REQ-RA-510: startServer 데몬 시작

시스템은 `startServer()` 함수를 export하여 임베딩 서버를 detached 백그라운드 프로세스로 시작해야 한다(SHALL).

#### Scenario: 데몬 프로세스 시작

- **GIVEN** 임베딩 서버가 실행 중이 아닌 상태
- **WHEN** `await startServer()`를 호출한다
- **THEN** `child_process.spawn('node', [serverPath], { detached: true, stdio: 'ignore' })`로 서버를 시작하고(SHALL), `child.unref()`로 부모 프로세스와 분리한다(SHALL). `serverPath`는 `~/.self-generation/lib/embedding-server.mjs`이다

#### Scenario: 호출자 프로세스 비차단

- **GIVEN** `startServer()`가 호출된 상태
- **WHEN** 서버 프로세스가 spawn된다
- **THEN** 부모 프로세스(훅)는 서버 프로세스 종료를 기다리지 않고 즉시 반환한다(SHALL). `detached: true`와 `child.unref()`로 보장한다

---

## 비기능 요구사항

### 성능

- 서버 모델 로딩은 최초 1회만 수행하며, 이후 요청은 ~5ms 이내에 처리되어야 한다(SHALL)
- 클라이언트 타임아웃은 10초로 설정하여 콜드 스타트(모델 로딩 대기) 시에도 응답을 보장해야 한다(SHALL)
- health check 타임아웃은 500ms로 설정한다(SHALL)

### 리소스 관리

- idle timeout 30분으로 미사용 시 자동 종료하여 메모리를 회수해야 한다(SHALL)
- 서버 프로세스는 Transformers.js 모델(~100MB RAM)을 메모리에 상주시킨다
- detached 프로세스로 실행하여 훅 프로세스 생명주기와 독립적이어야 한다(SHALL)

### 안정성

- 클라이언트 disconnect 에러는 무시하여 서버 안정성을 보장해야 한다(SHALL)
- 잘못된 JSON 입력에도 서버가 종료되지 않아야 한다(SHALL NOT)
- EADDRINUSE 감지로 다중 인스턴스를 방지해야 한다(SHALL)

---

## 제약사항

- 소켓 경로: `/tmp/self-gen-embed.sock` (하드코딩)
- 모델: `Xenova/paraphrase-multilingual-MiniLM-L12-v2` (384차원, 오프라인 지원)
- 모델 캐시: `~/.self-generation/models/`
- 프로토콜: newline-delimited JSON (한 연결에 한 요청)
- 서버는 ES Module (`.mjs`)로 작성, `node` 직접 실행

### 비고: config.json 연동

현재 DESIGN.md 참조 구현은 하드코딩된 상수(`SOCKET_PATH = '/tmp/self-gen-embed.sock'`, `IDLE_TIMEOUT_MS = 30 * 60 * 1000`, `TIMEOUT_MS = 10000`)를 사용한다. `config.json`의 `embedding.server` 섹션을 통한 설정 연동은 향후 구현 시 적용할 수 있다(MAY).

---

## 의존성

| 모듈 | import | 용도 |
|------|--------|------|
| `net` (Node.js) | `createServer`, `createConnection` | Unix socket 서버/클라이언트 |
| `@xenova/transformers` | `pipeline`, `env` | ML 모델 로딩 및 임베딩 생성 |
| `fs` (Node.js) | `existsSync`, `unlinkSync` | stale 소켓 파일 정리 |
| `child_process` (Node.js) | `spawn` | 서버 프로세스 시작 (클라이언트) |
| `path`, `os` (Node.js) | `join`, `homedir` | 경로 구성 |

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| 임베딩 데몬 | Transformers.js 모델을 상주 실행하는 Unix socket 서버 프로세스 (`embedding-server.mjs`) |
| 임베딩 클라이언트 | 훅에서 데몬과 통신하는 경량 모듈 (`embedding-client.mjs`) |
| 콜드 스타트 | 데몬 미실행 시 모델 로딩부터 시작하는 최초 응답 지연 (1~4초) |
| idle timeout | 마지막 요청 이후 30분 무활동 시 자동 종료 |
| stale socket | 이전 프로세스 비정상 종료로 남은 소켓 파일 |
| health check | `{"action":"health"}` 요청으로 서버 상태를 확인하는 프로토콜 |
| EADDRINUSE | 소켓 경로가 이미 사용 중일 때 발생하는 Node.js 에러 코드 |
| detached process | 부모 프로세스와 독립적으로 실행되는 백그라운드 프로세스 |
