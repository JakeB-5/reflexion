---
feature: session-summary
created: 2026-02-07
status: draft
---

# 구현 계획: session-summary

> SessionEnd 훅 — 세션 요약 집계 및 AI 분석 트리거

---

## 개요

`hooks/session-summary.mjs`는 세션 종료 시 트리거되는 훅 스크립트이다. 해당 세션의 모든 이벤트를 집계하여 요약 엔트리를 기록하고, 조건 충족 시 비동기 AI 분석을 트리거한다.

---

## 기술 결정

### 결정 1: 전체 파일 스캔 기반 집계

**근거:** Phase 1에서는 구현 단순성 우선. 대용량 최적화(세션 인덱스)는 필요 시 도입

**대안 검토:**
- 세션 인덱스 기반 조회 — 성능 우수하나 추가 인덱스 관리 필요
- 인메모리 집계 — 훅 간 상태 공유 불가 (각 훅은 독립 프로세스)

### 결정 2: AI 분석 비동기 트리거

**근거:** 분석은 10-30초 소요되므로 훅에서 동기 실행하면 세션 종료가 지연됨. `child_process.spawn`으로 detach 실행

---

## 구현 단계

### Phase 1: 스캐폴드

훅 스크립트 파일 및 의존성 설정

**산출물:**
- [ ] `~/.self-generation/hooks/session-summary.mjs` 파일 생성
- [ ] db.mjs import 설정

### Phase 2: 핵심 로직

세션 집계 및 AI 트리거 조건 구현

**산출물:**
- [ ] SessionSummaryEntry 스키마로 요약 기록
- [ ] sessionId 필터로 현재 세션 이벤트 조회
- [ ] promptCount, toolCounts, toolSequence, errorCount, uniqueErrors 집계
- [ ] AI 분석 트리거 조건 (promptCount >= 3 && reason !== 'clear')
- [ ] try-catch + exit(0) non-blocking 래퍼

### Phase 3: 테스트

집계 로직 및 트리거 조건 테스트

**산출물:**
- [ ] 정상 세션 요약 집계 검증
- [ ] toolCounts 집계 정확성 검증
- [ ] uniqueErrors 중복 제거 검증
- [ ] AI 트리거 조건 (3개 이상/미만) 검증
- [ ] clear 이유 세션 스킵 검증

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| 대용량 이벤트 조회 성능 | MEDIUM | SQL 인덱스 활용으로 최적화 |
| AI 분석 프로세스 spawn 실패 | LOW | try-catch로 요약 기록은 정상 완료 |
| 빈 세션 (이벤트 0개) | LOW | 0 값으로 정상 기록 |

---

## 테스트 전략

### 단위 테스트

- 집계 로직 (toolCounts, uniqueErrors) 독립 테스트
- AI 트리거 조건 분기 테스트
- 커버리지 목표: 80% 이상

### 통합 테스트

- 여러 이벤트 기록 후 세션 요약 생성 플로우
- SQLite `events` 테이블에서 요약 엔트리 확인

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] 작업 분해 (tasks.md)
3. [ ] 구현 시작
