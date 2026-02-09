---
feature: embedding-daemon
created: 2026-02-09
status: draft
---

# 구현 계획: embedding-daemon

> `lib/embedding-server.mjs` + `lib/embedding-client.mjs` — Unix socket 기반 임베딩 데몬 서버 및 클라이언트 구현 계획

---

## 개요

임베딩 데몬은 Transformers.js 모델을 메모리에 상주시켜 훅 프로세스가 매번 모델을 로딩하지 않고 ~5ms로 임베딩을 생성할 수 있게 하는 인프라 모듈이다. 서버와 클라이언트 두 파일로 구성되며, 서버는 독립 프로세스로, 클라이언트는 훅에서 import하여 사용한다.

---

## 기술 결정

### 결정 1: Unix Domain Socket 통신

**근거:** TCP 포트 대비 Unix socket은 로컬 전용이라 보안 위험이 없고, 파일 시스템 기반 존재 여부 확인이 용이하다. `/tmp/` 경로를 사용하여 OS 재시작 시 자동 정리된다.

**대안 검토:**
- HTTP 서버: 오버헤드 크고 포트 충돌 가능성
- Named pipe: 양방향 통신 복잡
- IPC (child_process fork): 훅 프로세스 생명주기와 결합되어 상주 불가

### 결정 2: Newline-Delimited JSON 프로토콜

**근거:** 한 연결에 한 요청 패턴으로 단순하고, JSON 파싱만으로 구현 가능. HTTP 같은 복잡한 프로토콜이 불필요하다.

**대안 검토:**
- Protocol Buffers: 외부 의존성 추가, 384차원 벡터에 과도
- MessagePack: 바이너리 디버깅 어려움

### 결정 3: Detached 프로세스 실행

**근거:** 훅은 2~5초 타임아웃 내에 종료되므로, 서버는 훅 프로세스와 독립적으로 실행되어야 한다. `detached: true` + `child.unref()`로 부모 종료 시에도 서버가 유지된다.

### 결정 4: 30분 Idle Timeout

**근거:** 세션 간 간격이 30분 이상이면 메모리(~100MB)를 회수하는 것이 합리적이다. 다음 세션 시작 시 `SessionStart` 훅에서 자동 재시작한다.

---

## 구현 단계

### Phase 1: 서버 스캐폴드 및 모델 로딩

서버 파일 생성, 모델 로딩, 기본 소켓 서버 구조

**산출물:**
- [ ] `lib/embedding-server.mjs` 파일 생성 — import, 상수 (SOCKET_PATH, IDLE_TIMEOUT_MS)
- [ ] `init()` 함수 구현 — env.cacheDir 설정 + pipeline 로딩
- [ ] `embed(texts)` 함수 구현 — 텍스트 배열 → 벡터 배열 변환, null/빈 텍스트 처리
- [ ] stale 소켓 정리 로직 (existsSync + unlinkSync)

### Phase 2: 서버 프로토콜 및 생명주기

소켓 통신, idle timeout, 에러 처리, graceful shutdown

**산출물:**
- [ ] `createServer()` + newline-delimited JSON 프로토콜 구현
- [ ] `handleRequest()` — embed, health, unknown action 분기
- [ ] `resetIdleTimer()` — 30분 idle timeout 관리
- [ ] EADDRINUSE 핸들러 구현
- [ ] SIGTERM/SIGINT graceful shutdown 핸들러

### Phase 3: 클라이언트 구현

소켓 클라이언트, 자동 시작, 상태 확인

**산출물:**
- [ ] `lib/embedding-client.mjs` 파일 생성
- [ ] `_sendRequest(texts)` 내부 함수 — 소켓 연결, JSON 송수신, 10초 타임아웃
- [ ] `embedViaServer(texts)` — auto-start + retry 로직
- [ ] `isServerRunning()` — health check (500ms 타임아웃)
- [ ] `startServer()` — detached spawn + unref

### Phase 4: 테스트

단위 테스트 및 통합 테스트

**산출물:**
- [ ] 서버 단위 테스트 — embed(), handleRequest(), idle timeout
- [ ] 클라이언트 단위 테스트 — _sendRequest(), embedViaServer() retry, isServerRunning()
- [ ] 통합 테스트 — 서버 시작 → 클라이언트 요청 → 응답 검증
- [ ] edge case — 동시 연결, stale 소켓, EADDRINUSE

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| 모델 로딩 시간(1~4초)으로 첫 요청 지연 | 🟡 MEDIUM | SessionStart 훅에서 선행 시작; 클라이언트 5초 대기 후 재시도 |
| stale 소켓 파일로 EADDRINUSE 오판 | 🟡 MEDIUM | 시작 전 existsSync + unlinkSync로 정리 |
| 메모리 누수(모델 ~100MB 상주) | 🟢 LOW | 30분 idle timeout으로 자동 회수 |
| 동시 시작 race condition | 🟢 LOW | EADDRINUSE 감지로 중복 인스턴스 방지 |
| /tmp 정리로 소켓 삭제 | 🟢 LOW | 클라이언트 auto-start로 자동 복구 |

---

## 테스트 전략

### 단위 테스트

- `embed()`: 정상 텍스트, 빈 텍스트, null, isFinite 실패
- `handleRequest()`: embed action, health action, unknown action, 잘못된 JSON
- `resetIdleTimer()`: 타이머 리셋, 만료 시 종료
- `_sendRequest()`: 정상 응답, 타임아웃, 에러 응답
- `embedViaServer()`: 정상, ECONNREFUSED 시 auto-start + retry
- `isServerRunning()`: 서버 실행 중/미실행/타임아웃
- `startServer()`: detached spawn 검증

### 통합 테스트

- 서버 시작 → 클라이언트 embed 요청 → 384차원 벡터 응답 검증
- 서버 미실행 → 클라이언트 auto-start → 재시도 성공
- idle timeout 후 → 클라이언트 재연결 → auto-start

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] Phase 1부터 순차 구현 시작
3. [ ] `data-collection/log-writer` REQ-DC-008 (`generateEmbeddings`)과의 통합 확인
4. [ ] `ai-analysis/session-start-hook` REQ-SSH-006 (데몬 자동 시작)과의 통합 확인
