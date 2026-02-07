---
feature: log-writer
created: 2026-02-07
status: draft
---

# 구현 계획: log-writer

> JSONL 읽기/쓰기 유틸리티 모듈 — 모든 데이터 수집 훅의 공통 기반 라이브러리

---

## 개요

`lib/log-writer.mjs`는 모든 훅 스크립트가 의존하는 핵심 유틸리티이다. JSONL 파일 I/O, 로그 로테이션, 보관 기한 관리, 설정 로딩, stdin 파싱 기능을 순수 Node.js 내장 모듈로 구현한다.

---

## 기술 결정

### 결정 1: JSONL (JSON Lines) 형식 사용

**근거:** Append-only 쓰기에 최적화, 외부 DB 불필요, 텍스트 도구로 직접 디버깅 가능

**대안 검토:**
- SQLite — 쿼리 유연하지만 외부 의존성 필요
- 단일 JSON 배열 — 동시 쓰기 시 파일 손상 위험

### 결정 2: 동기 I/O 사용 (readFileSync, appendFileSync)

**근거:** Hook은 단발 실행 프로세스로, 비동기 I/O의 이점이 없고 코드 복잡도만 증가

---

## 구현 단계

### Phase 1: 스캐폴드

디렉토리 구조와 모듈 파일 생성

**산출물:**
- [ ] `~/.self-generation/lib/log-writer.mjs` 파일 생성
- [ ] `~/.self-generation/data/` 디렉토리 자동 생성 로직

### Phase 2: 핵심 함수 구현

6개 export 함수와 2개 내부 함수 구현

**산출물:**
- [ ] `getLogFile()` — 로그 파일 경로 반환 + 디렉토리 자동 생성
- [ ] `getProjectName(cwd)` — cwd에서 프로젝트명 추출
- [ ] `appendEntry(logFile, entry)` — JSONL 엔트리 추가 + 로테이션 검사
- [ ] `readEntries(logFile, filterOrLimit)` — 필터/축약 조회
- [ ] `readStdin()` — stdin JSON 동기 파싱
- [ ] `loadConfig()` — config.json 로딩 + 기본값
- [ ] `rotateIfNeeded(logFile)` — 50MB 로테이션 (내부)
- [ ] `pruneOldLogs()` — 90일 초과 삭제 (내부)

### Phase 3: 테스트

단위 테스트 및 엣지 케이스 검증

**산출물:**
- [ ] appendEntry 정상/대용량 테스트
- [ ] readEntries 필터 조합 테스트
- [ ] 로테이션 임계값 테스트
- [ ] 설정 파일 부재 시 기본값 테스트
- [ ] TOCTOU 동시 접근 테스트

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| 동시 훅 실행 시 파일 손상 | MEDIUM | appendFileSync 원자성 활용, TOCTOU 에러 무시 |
| 대용량 JSONL 읽기 성능 | LOW | since 필터 최적화, 향후 세션 인덱스 도입 |
| config.json 파싱 실패 | LOW | 기본값 fallback으로 항상 동작 보장 |

---

## 테스트 전략

### 단위 테스트

- 각 export 함수별 독립 테스트
- 커버리지 목표: 80% 이상

### 통합 테스트

- 훅 스크립트에서 log-writer 함수 호출 시나리오
- 로테이션 후 새 파일에 정상 기록 확인

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] 작업 분해 (tasks.md)
3. [ ] 구현 시작
