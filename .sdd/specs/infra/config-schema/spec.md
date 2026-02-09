---
id: config-schema
title: "config-schema"
status: draft
created: 2026-02-09
domain: infra
depends: null
constitution_version: "2.0.0"
---

# Config Schema

> 시스템 설정 스키마 모듈. `~/.reflexion/config.json` 파일의 전체 스키마를 정의하고, 설정 로딩(`loadConfig`), 시스템 활성화 확인(`isEnabled`), 기본값 병합, 검증 규칙을 제공한다. 모든 훅과 CLI 도구가 이 모듈을 통해 설정에 접근한다.

---

## Requirement: REQ-INF-101 — config.json 전체 스키마 정의

시스템은 `~/.reflexion/config.json`의 스키마를 다음과 같이 정의(SHALL)해야 한다.

### 최상위 필드

| 필드 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `enabled` | boolean | `true` | 시스템 전체 활성화 여부 |
| `collectPromptText` | boolean | `true` | 프롬프트 원문 수집 여부. `false` 시 `[REDACTED]`로 대체 |
| `retentionDays` | number | `90` | 이벤트 보관 기한 (일) |
| `analysisModel` | string | `"claude-sonnet-4-5-20250929"` | AI 분석에 사용할 Claude 모델 ID |
| `analysisOnSessionEnd` | boolean | `true` | 세션 종료 시 AI 분석 자동 실행 여부 |
| `analysisDays` | number | `7` | AI 분석 대상 기간 (일) |
| `analysisCacheMaxAgeHours` | number | `24` | AI 분석 캐시 유효 기간 (시간) |
| `dbPath` | string | `"~/.reflexion/data/reflexion.db"` | SQLite DB 파일 경로 |
| `embedding` | object | *(하위 참조)* | 임베딩 설정 객체 |

### embedding 중첩 객체

| 필드 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `embedding.enabled` | boolean | `true` | 임베딩 기능 활성화 여부 |
| `embedding.model` | string | `"Xenova/paraphrase-multilingual-MiniLM-L12-v2"` | Transformers.js 모델 ID |
| `embedding.dimensions` | number | `384` | 임베딩 벡터 차원 수 |
| `embedding.threshold` | number | `0.76` | 벡터 유사도 매칭 임계값 |
| `embedding.batchSize` | number | `50` | 배치 임베딩 처리 단위 |
| `embedding.modelCacheDir` | string | `"~/.reflexion/models/"` | 모델 캐시 디렉토리 |

### embedding.server 중첩 객체

| 필드 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `embedding.server.socketPath` | string | `"/tmp/self-gen-embed.sock"` | Unix 소켓 경로 |
| `embedding.server.idleTimeoutMinutes` | number | `30` | 데몬 유휴 타임아웃 (분) |
| `embedding.server.clientTimeoutMs` | number | `10000` | 클라이언트 요청 타임아웃 (ms) |

### Scenario: 전체 config.json 예시

- **GIVEN** 사용자가 모든 기본값을 사용하는 상태
- **WHEN** `config.json`이 초기화되면
- **THEN** 다음 JSON 구조가 생성(SHALL)된다:
  ```json
  {
    "enabled": true,
    "collectPromptText": true,
    "retentionDays": 90,
    "analysisModel": "claude-sonnet-4-5-20250929",
    "analysisOnSessionEnd": true,
    "analysisDays": 7,
    "analysisCacheMaxAgeHours": 24,
    "dbPath": "~/.reflexion/data/reflexion.db",
    "embedding": {
      "enabled": true,
      "model": "Xenova/paraphrase-multilingual-MiniLM-L12-v2",
      "dimensions": 384,
      "threshold": 0.76,
      "batchSize": 50,
      "modelCacheDir": "~/.reflexion/models/",
      "server": {
        "socketPath": "/tmp/self-gen-embed.sock",
        "idleTimeoutMinutes": 30,
        "clientTimeoutMs": 10000
      }
    }
  }
  ```

---

## Requirement: REQ-INF-102 — 설정 로딩 (loadConfig)

시스템은 `~/.reflexion/config.json` 파일을 읽어 JavaScript 객체로 반환하는 `loadConfig()` 함수를 제공(SHALL)해야 한다.

### Scenario: 정상 설정 파일 로딩

- **GIVEN** `~/.reflexion/config.json`이 유효한 JSON으로 존재하는 상태
- **WHEN** `loadConfig()`를 호출하면
- **THEN** 파일 내용을 `JSON.parse()`로 파싱하여 객체를 반환(SHALL)한다

### Scenario: config.json 미존재 시 기본값 반환

- **GIVEN** `~/.reflexion/config.json` 파일이 존재하지 않는 상태
- **WHEN** `loadConfig()`를 호출하면
- **THEN** 빈 객체 `{}`를 반환(SHALL)한다. 각 사용처에서 필드별 기본값을 적용한다

### Scenario: JSON 파싱 에러 시 기본값 반환

- **GIVEN** `config.json` 파일이 존재하지만 유효하지 않은 JSON인 상태
- **WHEN** `loadConfig()`를 호출하면
- **THEN** try-catch로 파싱 에러를 포착하고 빈 객체 `{}`를 반환(SHALL)한다. 시스템은 기본값으로 정상 동작을 보장한다

### Scenario: 경로 해석

- **GIVEN** `loadConfig()` 내부에서 파일 경로를 구성하는 상태
- **WHEN** 경로를 결정하면
- **THEN** `GLOBAL_DIR` 상수(`~/.reflexion/`)와 `'config.json'`을 `path.join()`으로 결합(SHALL)한다

---

## Requirement: REQ-INF-103 — 시스템 활성화 확인 (isEnabled)

시스템은 `config.enabled` 필드를 확인하여 시스템 활성화 여부를 반환하는 `isEnabled()` 함수를 제공(SHALL)해야 한다.

### Scenario: enabled: false 시 비활성화

- **GIVEN** `config.json`에 `"enabled": false`가 설정된 상태
- **WHEN** `isEnabled()`를 호출하면
- **THEN** `false`를 반환(SHALL)한다. 각 훅은 이 반환값에 따라 `process.exit(0)`으로 즉시 종료한다

### Scenario: enabled 필드 미지정 시 기본 활성화

- **GIVEN** `config.json`이 없거나 `enabled` 필드가 명시되지 않은 상태
- **WHEN** `isEnabled()`를 호출하면
- **THEN** `true`를 반환(SHALL)한다. `config.enabled !== false` 비교로 기본 활성화를 보장한다

### Scenario: 훅에서의 사용 패턴

- **GIVEN** 모든 훅 스크립트가 실행을 시작하는 상태
- **WHEN** 훅의 메인 함수 첫 줄에서 `isEnabled()`를 호출하면
- **THEN** `false` 반환 시 즉시 `process.exit(0)`으로 종료(SHALL)하여 비활성화 상태에서 불필요한 처리를 방지한다

---

## Requirement: REQ-INF-104 — 필드별 기본값 상수

시스템은 각 설정 필드의 기본값을 모듈 상수로 정의(SHALL)해야 한다. `loadConfig()`가 빈 객체를 반환할 때 각 사용처에서 이 상수를 참조한다.

### Scenario: 기본값 상수 정의

- **GIVEN** `config-schema` 모듈이 로드된 상태
- **WHEN** 기본값이 필요한 사용처에서 상수를 참조하면
- **THEN** 다음 상수가 정의(SHALL)되어 있다:
  - `RETENTION_DAYS = 90`
  - `ANALYSIS_DAYS = 7`
  - `ANALYSIS_CACHE_MAX_AGE_HOURS = 24`
  - `DEFAULT_DB_PATH = '~/.reflexion/data/reflexion.db'`
  - `DEFAULT_EMBEDDING_MODEL = 'Xenova/paraphrase-multilingual-MiniLM-L12-v2'`
  - `DEFAULT_EMBEDDING_DIMENSIONS = 384`
  - `DEFAULT_EMBEDDING_THRESHOLD = 0.76`
  - `DEFAULT_BATCH_SIZE = 50`
  - `DEFAULT_SOCKET_PATH = '/tmp/self-gen-embed.sock'`
  - `DEFAULT_IDLE_TIMEOUT_MINUTES = 30`
  - `DEFAULT_CLIENT_TIMEOUT_MS = 10000`

### Scenario: 부분 설정 시 기본값 병합

- **GIVEN** `config.json`에 `{ "retentionDays": 30 }`만 설정된 상태
- **WHEN** 훅에서 `loadConfig()`를 호출하고 `config.analysisDays`를 참조하면
- **THEN** `config.analysisDays || ANALYSIS_DAYS`로 기본값 `7`이 적용(SHALL)된다

---

## Requirement: REQ-INF-105 — 필드 타입 검증 규칙

시스템은 설정 필드의 타입 및 값 범위를 다음과 같이 검증(SHOULD)해야 한다. 검증 실패 시 기본값으로 대체한다.

### 검증 규칙 테이블

| 필드 | 타입 제약 | 값 범위 | 기본값 대체 |
|------|----------|---------|------------|
| `enabled` | `typeof === 'boolean'` | — | `true` |
| `collectPromptText` | `typeof === 'boolean'` | — | `true` |
| `retentionDays` | `typeof === 'number'` | `> 0` | `90` |
| `analysisOnSessionEnd` | `typeof === 'boolean'` | — | `true` |
| `analysisDays` | `typeof === 'number'` | `> 0` | `7` |
| `analysisCacheMaxAgeHours` | `typeof === 'number'` | `> 0` | `24` |
| `dbPath` | `typeof === 'string'` | 비어있지 않음 | `"~/.reflexion/data/reflexion.db"` |
| `embedding` | `typeof === 'object'` | `null`이 아님 | `{}` |
| `embedding.dimensions` | `typeof === 'number'` | `> 0` | `384` |
| `embedding.threshold` | `typeof === 'number'` | `0 < x ≤ 1` | `0.76` |
| `embedding.batchSize` | `typeof === 'number'` | `> 0` | `50` |
| `embedding.server.idleTimeoutMinutes` | `typeof === 'number'` | `> 0` | `30` |
| `embedding.server.clientTimeoutMs` | `typeof === 'number'` | `> 0` | `10000` |

### Scenario: 잘못된 타입의 retentionDays

- **GIVEN** `config.json`에 `"retentionDays": "abc"`가 설정된 상태
- **WHEN** `loadConfig()` 후 `config.retentionDays || RETENTION_DAYS`를 평가하면
- **THEN** falsy 평가에 의해 기본값 `90`이 적용(SHOULD)된다

### Scenario: 음수 값의 retentionDays

- **GIVEN** `config.json`에 `"retentionDays": -5`가 설정된 상태
- **WHEN** 설정값을 사용하려고 하면
- **THEN** 사용처에서 `config.retentionDays > 0 ? config.retentionDays : RETENTION_DAYS`로 기본값을 적용(SHOULD)한다

### Scenario: embedding 필드가 문자열인 경우

- **GIVEN** `config.json`에 `"embedding": "invalid"`가 설정된 상태
- **WHEN** `config.embedding?.model`을 참조하면
- **THEN** 옵셔널 체이닝에 의해 `undefined`가 반환되고, 기본값 `DEFAULT_EMBEDDING_MODEL`이 적용(SHOULD)된다

---

## Requirement: REQ-INF-106 — install-cli 초기 설정 생성

시스템은 `install.mjs` 실행 시 `config.json`이 없으면 최소 설정으로 초기화(SHALL)해야 한다.

### Scenario: 최초 설치 시 config.json 생성

- **GIVEN** `~/.reflexion/config.json`이 존재하지 않는 상태
- **WHEN** `install.mjs`가 실행되면
- **THEN** 다음 최소 설정이 생성(SHALL)된다:
  ```json
  {
    "enabled": true,
    "collectPromptText": true,
    "retentionDays": 90,
    "analysisModel": "claude-sonnet-4-5-20250929"
  }
  ```

### Scenario: 기존 config.json 보존

- **GIVEN** `~/.reflexion/config.json`이 이미 존재하는 상태
- **WHEN** `install.mjs`가 실행되면
- **THEN** 기존 파일을 덮어쓰지 않고 보존(SHALL)한다

---

## 비고

- `loadConfig()`와 `isEnabled()`는 `lib/db.mjs`에서 export되는 유틸리티 함수로, 모든 훅 스크립트가 공통으로 사용한다
- `loadConfig()`는 동기 함수이다. `readFileSync()`와 `existsSync()`를 사용하여 Hook의 빠른 실행을 보장한다
- 이 스펙은 config.json의 **스키마와 접근 인터페이스**를 정의하며, 실제 함수 구현은 `log-writer` 스펙(REQ-DC-005)에 포함된다
- DESIGN.md의 install-cli 코드에서 `analysisModel` 필드가 초기 생성 시 포함되지만, 9.6절 전체 스키마에는 명시되지 않았다. 구현 시 이 필드도 스키마에 포함할 것을 권장(SHOULD)한다
- `embedding` 객체는 선택적(OPTIONAL)이다. 미지정 시 모든 임베딩 관련 기능은 기본값으로 동작한다
- 틸드(`~`)로 시작하는 경로(`dbPath`, `modelCacheDir`)는 런타임에 `os.homedir()`로 확장된다
