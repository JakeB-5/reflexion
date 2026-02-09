---
feature: log-writer
created: 2026-02-07
status: draft
---

# 구현 계획: log-writer

> SQLite DB 관리 유틸리티 모듈 — 모든 데이터 수집 훅의 공통 기반 라이브러리

---

## 개요

`lib/db.mjs`는 모든 훅 스크립트가 의존하는 핵심 유틸리티이다. SQLite DB 연결, CRUD, 벡터 검색 유틸리티, 설정 로딩, stdin 파싱 기능을 `better-sqlite3` 기반으로 구현한다.

---

## 기술 결정

### 결정 1: SQLite (`better-sqlite3`) 사용

**근거:** 구조화된 쿼리, 동시 접근 안전(WAL 모드), `sqlite-vec` 확장으로 벡터 검색 지원

**대안 검토:**
- JSONL — Append-only 단순하지만 쿼리/필터링 비효율
- 단일 JSON 배열 — 동시 쓰기 시 파일 손상 위험

### 결정 2: 동기 I/O 사용 (`better-sqlite3` 동기 API)

**근거:** Hook은 단발 실행 프로세스로, `better-sqlite3`의 동기 API가 코드 단순성과 성능 모두 최적

---

## 구현 단계

### Phase 1: 스캐폴드

디렉토리 구조와 모듈 파일 생성

**산출물:**
- [ ] `~/.self-generation/lib/db.mjs` 파일 생성
- [ ] `~/.self-generation/data/` 디렉토리 자동 생성 로직

### Phase 2: 핵심 함수 구현

6개 export 함수와 2개 내부 함수 구현

**산출물:**
- [ ] `getDb()` — SQLite DB 연결 반환 + DB 파일/디렉토리 자동 생성
- [ ] `getProjectName(cwd)` — cwd에서 프로젝트명 추출
- [ ] `insertEvent(entry)` — `events` 테이블에 이벤트 INSERT
- [ ] `queryEvents(filterOrLimit)` — 필터/축약 조회
- [ ] `readStdin()` — stdin JSON 동기 파싱
- [ ] `loadConfig()` — config.json 로딩 + 기본값
- [ ] `pruneOldEvents()` — 확률적 정리 (내부)

### Phase 3: 테스트

단위 테스트 및 엣지 케이스 검증

**산출물:**
- [ ] insertEvent 정상/대용량 테스트
- [ ] queryEvents 필터 조합 테스트
- [ ] 설정 파일 부재 시 기본값 테스트
- [ ] WAL 모드 동시 접근 테스트

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| 동시 훅 실행 시 DB 충돌 | MEDIUM | WAL 모드로 동시 읽기/쓰기 지원 |
| 대용량 이벤트 조회 성능 | LOW | SQL 인덱스 활용, 필터 조건 최적화 |
| config.json 파싱 실패 | LOW | 기본값 fallback으로 항상 동작 보장 |

---

## 테스트 전략

### 단위 테스트

- 각 export 함수별 독립 테스트
- 커버리지 목표: 80% 이상

### 통합 테스트

- 훅 스크립트에서 db.mjs 함수 호출 시나리오
- pruneOldEvents 후 정상 기록 확인

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] 작업 분해 (tasks.md)
3. [ ] 구현 시작
