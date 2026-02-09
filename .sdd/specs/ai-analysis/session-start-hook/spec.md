---
id: session-start-hook
title: "SessionStart 캐시 주입 훅"
status: draft
created: 2026-02-07
domain: ai-analysis
depends: "ai-analysis/ai-analyzer, data-collection/log-writer, realtime-assist/embedding-daemon"
constitution_version: "2.0.0"
---

# SessionStart 캐시 주입 훅 (session-start-hook)

> SessionStart 이벤트에서 `analysis_cache` 테이블의 캐시된 AI 분석 결과 제안과 `events` 테이블의 이전 세션 컨텍스트를 `additionalContext` stdout으로 주입하고, 임베딩 데몬을 자동 시작하는 훅 스크립트 (`hooks/session-analyzer.mjs`). DB 접근은 `lib/db.mjs`를 통한다.

---

## 요구사항

### REQ-SSH-001: 시스템 활성화 확인

시스템은 `isEnabled()` 함수로 Self-Generation 시스템 활성화 여부를 확인하고, 비활성 시 즉시 exit code 0으로 종료해야 한다(SHALL).

#### Scenario: 시스템 비활성 시 즉시 종료

- **GIVEN** `config.json`에서 시스템이 비활성화되어 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `isEnabled()`가 `false`를 반환하고 아무 출력 없이 exit code 0으로 종료된다

#### Scenario: 시스템 활성 시 계속 실행

- **GIVEN** `config.json`에서 시스템이 활성화되어 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `isEnabled()`가 `true`를 반환하고 캐시 주입 로직을 계속 실행한다

---

### REQ-SSH-002: 캐시된 분석 결과 주입

시스템은 SessionStart 시 `getCachedAnalysis(24, project)`를 호출하여 24시간 이내의 프로젝트별 캐시된 분석 결과를 조회하고, 제안이 존재하면 최대 3개를 `additionalContext`로 주입해야 한다(SHALL). `project` 파라미터는 stdin의 `cwd`에서 `getProjectPath()` → `getProjectName()`으로 추출한다 (v9: 프로젝트별 캐시 오염 방지).

#### Scenario: 유효한 캐시에서 제안 주입

- **GIVEN** `analysis_cache` 테이블에 프로젝트 `'my-app'`의 2시간 전 분석 결과가 저장되어 있고, suggestions 5개가 포함되어 있다
- **WHEN** SessionStart 훅이 `cwd`가 `my-app` 프로젝트인 상태로 실행된다
- **THEN** `contextParts` 배열에 상위 3개 제안의 type, summary, id가 포함된 메시지가 추가된다

#### Scenario: 캐시 없음 시 제안 미주입

- **GIVEN** `analysis_cache` 테이블이 비어 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** 제안 관련 컨텍스트가 `contextParts`에 추가되지 않는다

#### Scenario: 캐시 만료 시 제안 미주입

- **GIVEN** `analysis_cache` 테이블에 30시간 전 분석 결과만 저장되어 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `getCachedAnalysis(24, project)`가 `null`을 반환하고 제안이 주입되지 않는다

---

### REQ-SSH-003: 제안 메시지 포맷팅

시스템은 제안을 사용자에게 표시할 형태로 포맷팅해야 한다(SHALL). 각 제안은 `- [type] summary [id: suggest-N]` 형식이어야 하며(SHALL), 적용(`apply.mjs`) 및 거부(`dismiss.mjs`) CLI 명령어 안내를 포함해야 한다(SHALL).

#### Scenario: 제안 포맷팅

- **GIVEN** suggestions 배열에 `{ type: 'skill', summary: 'TS 프로젝트 초기화 스킬', id: 'suggest-0' }` 항목이 있다
- **WHEN** 제안 메시지를 포맷팅한다
- **THEN** 출력 문자열에 다음이 포함된다:
  - `'[Self-Generation] AI 패턴 분석 결과:'` 헤더
  - `'- [skill] TS 프로젝트 초기화 스킬 [id: suggest-0]'` 제안 항목
  - `'node ~/.self-generation/bin/apply.mjs <번호>'` 적용 안내
  - `'node ~/.self-generation/bin/dismiss.mjs <id>'` 거부 안내

---

### REQ-SSH-004: 이전 세션 컨텍스트 주입

시스템은 `queryEvents({ type: 'session_summary', projectPath, limit: 1 })`로 `events` 테이블에서 가장 최근 `session_summary` 레코드를 조회하여 이전 세션의 컨텍스트를 주입해야 한다(SHOULD). 주입 정보에는 프롬프트 수, 도구 사용 횟수, 마지막 프롬프트, 편집 중이던 파일, 미해결 에러, 주요 도구 상위 3개를 포함해야 한다(SHOULD).

#### Scenario: 이전 세션 요약 주입

- **GIVEN** `events` 테이블에 이전 세션의 `type = 'session_summary'` 레코드가 존재하고, `promptCount: 15`, `toolCounts: { Read: 30, Edit: 20, Bash: 10 }`, `lastPrompts: ['테스트 작성해줘']`, `lastEditedFiles: ['src/app.ts']`, `errorCount: 2`, `uniqueErrors: ['TypeError: ...', 'RangeError: ...']` 필드가 포함되어 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `contextParts`에 다음 정보가 포함된 메시지가 추가된다:
  - `'[Self-Generation] 이전 세션 컨텍스트 ({ts}):'` 헤더
  - `'프롬프트 15개, 도구 60회 사용'`
  - `'이전 세션 마지막 작업: "테스트 작성해줘"'`
  - `'수정 중이던 파일: src/app.ts'`
  - `'미해결 에러 2건: TypeError: ..., RangeError: ...'`
  - `'주요 도구: Read(30), Edit(20), Bash(10)'`

#### Scenario: 이전 세션 요약 없음

- **GIVEN** `events` 테이블에 `type = 'session_summary'` 레코드가 없다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** 이전 세션 컨텍스트 없이 제안만 주입되거나 (제안도 없으면) 무출력 종료된다

---

### REQ-SSH-005: 세션 재개 감지 (Resume)

시스템은 stdin의 `source` 필드가 `'resume'`인 경우 세션 재개로 판단하고, 미해결 에러 정보를 `[RESUME]` 태그와 함께 더 상세하게 주입해야 한다(SHOULD).

#### Scenario: 재개 세션에서 미해결 에러 상세 주입

- **GIVEN** SessionStart stdin에 `source: 'resume'`이 포함되고, 이전 세션 요약에 `uniqueErrors: ['TypeError: x is not a function', 'SyntaxError: ...']`가 있다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `contextParts`에 `'[RESUME] 미해결 에러 상세: TypeError: x is not a function, SyntaxError: ...'`가 포함된다

#### Scenario: 일반 세션 시작 (비재개)

- **GIVEN** SessionStart stdin에 `source: 'startup'`이 포함된다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `[RESUME]` 태그는 포함되지 않고 일반 컨텍스트만 주입된다

---

### REQ-SSH-006: 임베딩 데몬 자동 시작

시스템은 SessionStart 시 임베딩 데몬(`embedding-server.mjs`)의 실행 상태를 확인하고, 실행 중이 아니면 자동으로 시작해야 한다(SHOULD). 임베딩 데몬 시작 실패는 무시해야 한다(SHALL) — 임베딩 데몬은 선택적 기능이다.

#### Scenario: 임베딩 데몬 미실행 시 자동 시작

- **GIVEN** 임베딩 데몬이 실행 중이 아니다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `isServerRunning()`이 `false`를 반환하고 `startServer()`가 호출되어 임베딩 데몬이 시작된다

#### Scenario: 임베딩 데몬 이미 실행 중

- **GIVEN** 임베딩 데몬이 이미 실행 중이다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** `isServerRunning()`이 `true`를 반환하고 `startServer()`가 호출되지 않는다

#### Scenario: 임베딩 데몬 시작 실패

- **GIVEN** `embedding-client.mjs` 모듈을 로드할 수 없거나 시작에 실패한다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** 예외가 catch되고 무시되며 나머지 훅 로직은 정상 실행된다

**구현 참조**:
```javascript
try {
  const { isServerRunning, startServer } = await import('../lib/embedding-client.mjs');
  if (!await isServerRunning()) {
    await startServer();
  }
} catch { /* Embedding daemon optional */ }
```

---

### REQ-SSH-007: stdout 출력 및 종료

시스템은 `contextParts` 배열에 내용이 있으면 `'\n\n'`으로 연결하여 `hookSpecificOutput.additionalContext`로 stdout에 JSON 출력해야 한다(SHALL). `contextParts`가 비어 있으면 stdout에 아무것도 출력하지 않아야 한다(SHALL). 모든 경우에 exit code 0으로 종료해야 한다(SHALL).

#### Scenario: 컨텍스트 존재 시 JSON 출력

- **GIVEN** `contextParts` 배열에 제안 메시지와 이전 세션 컨텍스트 두 항목이 있다
- **WHEN** 출력 단계에 도달한다
- **THEN** stdout으로 다음 JSON이 출력된다:
  ```json
  {
    "hookSpecificOutput": {
      "hookEventName": "SessionStart",
      "additionalContext": "<제안 메시지>\n\n<이전 세션 컨텍스트>"
    }
  }
  ```

#### Scenario: 컨텍스트 없을 시 무출력

- **GIVEN** `contextParts` 배열이 비어 있다
- **WHEN** 출력 단계에 도달한다
- **THEN** stdout에 아무것도 출력되지 않고 exit code 0으로 종료된다

---

### REQ-SSH-008: 훅 실패 안전성

시스템은 어떤 예외가 발생하더라도 exit code 0으로 종료해야 한다(SHALL). SessionStart 훅 실패가 Claude Code 세션 시작을 방해해서는 안 된다(SHALL NOT). 최상위 try-catch 블록으로 모든 예외를 포착해야 한다(SHALL).

#### Scenario: DB 손상 또는 접근 실패

- **GIVEN** `self-gen.db` 파일이 손상되었거나 접근할 수 없다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** 예외가 catch되고 exit code 0으로 종료된다

#### Scenario: 모듈 로드 실패

- **GIVEN** `db.mjs` 또는 `ai-analyzer.mjs` 모듈이 존재하지 않거나 import에 실패한다
- **WHEN** SessionStart 훅이 실행된다
- **THEN** 최상위 try-catch에서 예외가 처리되고 exit code 0으로 종료된다

---

## 비기능 요구사항

### 성능

- SessionStart 훅 전체 실행 시간은 100ms 이내여야 한다(SHALL)
- DB 조회만 수행하며, AI 호출이나 무거운 연산을 수행해서는 안 된다(SHALL NOT)
- 임베딩 데몬 시작은 비동기로 수행하며 훅 응답 시간에 영향을 주지 않아야 한다(SHOULD)

### 안정성

- 모든 코드 경로에서 exit code 0을 보장해야 한다(SHALL)
- 최상위 try-catch 블록으로 모든 예외를 포착해야 한다(SHALL)

---

## 제약사항

- stdin으로 `{ source, model, session_id, cwd }` 형태의 JSON을 받는다
- stdout JSON 형식: `{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "<문자열>" } }`
- Claude Code Hooks API 규격을 준수해야 한다

---

## 의존성

| 모듈 | import | 용도 |
|------|--------|------|
| `lib/db.mjs` | `queryEvents`, `getProjectName`, `getProjectPath`, `readStdin`, `isEnabled` | DB 조회, 프로젝트 경로 추출, stdin 파싱, 활성화 확인 |
| `lib/ai-analyzer.mjs` | `getCachedAnalysis` | 캐시된 분석 결과 조회 |
| `lib/embedding-client.mjs` | `isServerRunning`, `startServer` | 임베딩 데몬 상태 확인 및 시작 (dynamic import) |

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| `hooks/session-analyzer.mjs` | SessionStart 이벤트 훅 스크립트 |
| `additionalContext` | 훅이 Claude에게 주입하는 컨텍스트 문자열 |
| `contextParts` | 주입할 컨텍스트 조각들을 모으는 배열, `'\n\n'`으로 연결하여 최종 출력 |
| `source` | SessionStart stdin의 세션 시작 유형 필드 (`startup`, `resume`, `clear`, `compact`) |
| `[RESUME]` | 세션 재개 시 미해결 에러에 붙는 태그 |
| 임베딩 데몬 | Transformers.js 모델을 상주 실행하는 Unix socket 서버 프로세스 |
