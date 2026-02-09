---
feature: config-schema
created: 2026-02-09
status: draft
---

# 구현 계획: config-schema

> 시스템 설정 스키마 정의 및 설정 접근 인터페이스 — config.json의 전체 구조, 기본값, 검증 규칙

---

## 개요

`config-schema`는 `~/.reflexion/config.json`의 전체 스키마를 중앙 정의한다. 실제 `loadConfig()`와 `isEnabled()` 함수 구현은 `lib/db.mjs`(log-writer 스펙)에 위치하며, 이 스펙은 스키마 계약과 기본값 상수를 정의하는 역할을 한다.

---

## 기술 결정

### 결정 1: 스키마 정의만 분리, 구현은 db.mjs에 유지

**근거:** `loadConfig()`와 `isEnabled()`는 이미 `lib/db.mjs`의 export 함수로 설계되어 있다(DESIGN.md 4.2절). 별도 모듈로 분리하면 모든 훅의 import 경로가 변경되므로, 스키마 계약만 이 스펙에서 정의하고 구현은 기존 위치를 유지한다.

**대안 검토:**
- `lib/config.mjs` 별도 모듈 — import 경로 변경으로 모든 훅 수정 필요, 과도한 분리
- `lib/db.mjs` 내 인라인 — 현재 DESIGN.md 구조 그대로, 채택

### 결정 2: 기본값은 OR 연산자(`||`) 패턴

**근거:** DESIGN.md의 `pruneOldEvents()`가 `config.retentionDays || RETENTION_DAYS` 패턴을 사용. 별도의 deep-merge나 JSON Schema 검증 라이브러리 없이 최소 의존성으로 구현한다.

**대안 검토:**
- JSON Schema + ajv — 외부 의존성 추가 필요, 과도한 복잡도
- deep-merge 유틸 — 중첩 객체에 유용하나 3개 필드에 불필요

---

## 구현 단계

### Phase 1: 기본값 상수 정의

`lib/db.mjs` 상단에 config 관련 상수 집중 정의

**산출물:**
- [ ] 11개 기본값 상수 정의 (`RETENTION_DAYS`, `ANALYSIS_DAYS` 등)
- [ ] `GLOBAL_DIR` 경로 상수 확인

### Phase 2: loadConfig 보강

기존 `loadConfig()` 함수에 JSON 파싱 에러 핸들링 추가

**산출물:**
- [ ] try-catch 래핑으로 파싱 에러 시 `{}` 반환
- [ ] `existsSync` 체크 유지

### Phase 3: install-cli 초기 설정

`bin/install.mjs`에서 config.json 최초 생성 로직 구현

**산출물:**
- [ ] 최소 4개 필드로 초기 config.json 생성
- [ ] 기존 파일 존재 시 skip 로직

### Phase 4: 테스트

설정 로딩 및 기본값 적용 시나리오 검증

**산출물:**
- [ ] config.json 미존재 시 `{}` 반환 테스트
- [ ] JSON 파싱 에러 시 `{}` 반환 테스트
- [ ] `isEnabled()` true/false 분기 테스트
- [ ] 부분 설정 시 기본값 병합 테스트

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| config.json 파싱 실패로 전체 훅 중단 | HIGH | try-catch + `{}` fallback으로 항상 동작 보장 |
| embedding 중첩 객체 접근 시 TypeError | MEDIUM | 옵셔널 체이닝(`?.`)과 기본값 OR 패턴 사용 |
| install-cli가 기존 설정 덮어쓰기 | MEDIUM | `existsSync()` 체크로 기존 파일 보존 |

---

## 테스트 전략

### 단위 테스트

- `loadConfig()` 반환값 검증 (정상, 미존재, 파싱에러)
- `isEnabled()` 분기 검증
- 기본값 상수 존재 여부
- 커버리지 목표: 80% 이상

### 통합 테스트

- 훅에서 `isEnabled()` → `process.exit(0)` 흐름
- `pruneOldEvents()`에서 `loadConfig().retentionDays` 기본값 적용

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] 작업 분해 (tasks.md)
3. [ ] 구현 시작 (log-writer 구현과 병행)
