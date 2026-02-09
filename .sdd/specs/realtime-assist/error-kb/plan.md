---
feature: error-kb
created: 2026-02-07
status: draft
---

# 구현 계획: error-kb

> `lib/error-kb.mjs` — 에러 지식 베이스 유틸리티 모듈 구현 계획

---

## 개요

에러 KB 모듈은 에러 메시지 정규화, KB 검색, 해결 이력 기록 3가지 핵심 기능을 제공한다. `normalizeError()`의 단일 소유자(Single Owner)로서 다른 모듈이 import하여 사용하는 기반 유틸리티 역할을 한다.

---

## 기술 결정

### 결정 1: 동기 SQLite 사용 (better-sqlite3)

**근거:** 훅 환경에서 프로세스가 즉시 종료되므로, 비동기 I/O가 완료 전에 프로세스가 끝날 위험이 있다. better-sqlite3의 동기 API로 기록 완료를 보장한다.

**대안 검토:**
- async/await + 비동기 DB: 프로세스 종료 전 기록 누락 위험
- Worker Threads: 오버엔지니어링, 2s 타임아웃 내 불필요

### 결정 2: 역순 검색 (최신 우선)

**근거:** 동일 에러의 최신 해결법이 가장 관련성 높으므로, 파일을 역순으로 읽어 최신 엔트리를 먼저 반환한다.

---

## 구현 단계

### Phase 1: 스캐폴드 및 정규화 함수

모듈 파일 생성, 상수 정의, `normalizeError()` 구현

**산출물:**
- [ ] `lib/error-kb.mjs` 파일 생성 (KB_FILE 상수, import)
- [ ] `normalizeError()` 구현 — 정규식 치환 (PATH, N, STR) + 200자 절단 + trim
- [ ] `normalizeError()` 단위 테스트

### Phase 2: 핵심 기능 (검색 + 기록)

KB 검색(`searchErrorKB`)과 해결 기록(`recordResolution`) 구현

**산출물:**
- [ ] `searchErrorKB()` 구현 — exact match → substring fallback, useCount 증가
- [ ] `recordResolution()` 구현 — ErrorKBEntry append
- [ ] 검색/기록 단위 테스트

### Phase 3: 통합 테스트

다른 모듈과의 통합 및 edge case 테스트

**산출물:**
- [ ] KB 파일 부재 시 동작 테스트
- [ ] 대용량 KB (10,000+라인) 성능 테스트
- [ ] 잘못된 JSON 라인 스킵 테스트
- [ ] error-logger.mjs 훅 확장에서 import 테스트

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| 대용량 KB 파일로 검색 성능 저하 | 🟡 MEDIUM | 역순 검색으로 최신 매치 빠르게 반환; 필요시 인덱스 도입 |
| 정규화 정규식이 일부 에러 패턴 누락 | 🟢 LOW | 실사용 후 정규식 패턴 점진적 보완 |
| 동시 쓰기 시 DB 충돌 | 🟡 MEDIUM | SQLite WAL 모드로 동시 접근 지원, 실질적 위험 낮음 |

---

## 테스트 전략

### 단위 테스트

- `normalizeError()`: 경로/숫자/문자열 치환, 200자 절단, 빈 입력
- `searchErrorKB()`: 정확 매치, substring fallback, 파일 부재, 빈 파일
- `recordResolution()`: 정상 기록, 파일 자동 생성

### 통합 테스트

- error-logger.mjs에서 `normalizeError()` + `searchErrorKB()` 호출 흐름
- pre-tool-guide.mjs에서 `searchErrorKB()` 호출 흐름

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] Phase 1부터 순차 구현 시작
3. [ ] error-logger.mjs 훅 확장 (data-collection 도메인과 조율)
