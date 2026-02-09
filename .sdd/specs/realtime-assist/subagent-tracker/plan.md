---
feature: subagent-tracker
created: 2026-02-07
status: draft
---

# 구현 계획: subagent-tracker

> `hooks/subagent-tracker.mjs` — SubagentStop 이벤트 추적 훅 구현 계획

---

## 개요

subagent-tracker는 SubagentStop 이벤트를 수신하여 이벤트 로그와 성능 통계를 기록하는 단순한 데이터 수집 훅이다. stdout 출력은 하지 않고(컨텍스트 주입 없음), 데이터만 축적한다.

---

## 기술 결정

### 결정 1: 이중 기록 (events 테이블 + subagent_stop 타입 쿼리)

**근거:** `events` 테이블은 전체 이벤트 로그(배치 분석용), `subagent_stop` 타입 쿼리는 에이전트별 성능 통계(pre-tool-guide 소비용)로 역할 분리.

**대안 검토:**
- events 테이블만 사용: pre-tool-guide에서 전체 로그 스캔 필요, 성능 비효율
- subagent_stop 쿼리만 사용: 배치 분석에서 서브에이전트 이벤트 누락

### 결정 2: 성공/실패 판정 로직

**근거:** stdin 데이터에서 에러 필드 유무로 간단 판정. 복잡한 판정 로직은 향후 확장 시 추가.

---

## 구현 단계

### Phase 1: 스캐폴드 및 이벤트 기록

훅 스크립트 파일 생성, SubagentStopEntry 기록

**산출물:**
- [ ] `hooks/subagent-tracker.mjs` 파일 생성 (import, try-catch 구조)
- [ ] stdin 읽기 + SubagentStopEntry를 `events` 테이블에 insertEvent()
- [ ] 기본 동작 테스트 (정상 입력, 비정상 입력)

### Phase 2: 성능 통계 집계

`events` 테이블에 `subagent_stop` 타입으로 통계 엔트리 기록

**산출물:**
- [ ] SubagentStatsEntry 스키마 정의 및 insertEvent() 로직
- [ ] 성공/실패 판정 로직 구현
- [ ] 통계 기록 테스트

### Phase 3: 통합 테스트

Claude Code 훅 환경과의 통합 검증

**산출물:**
- [ ] exit 0 보장 테스트 (다양한 에러 시나리오)
- [ ] pre-tool-guide에서 `events` 테이블 `subagent_stop` 타입 쿼리 소비 테스트
- [ ] 훅 등록 JSON 검증

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| stdin 데이터 포맷 변경 | 🟡 MEDIUM | try-catch로 흡수, 로그에 가용 필드만 기록 |
| `events` 테이블 데이터 증가 | 🟢 LOW | 배치 분석에서 주기적 정리; pre-tool-guide는 최근 20건만 쿼리 |

---

## 테스트 전략

### 단위 테스트

- stdin 파싱: 정상 JSON, 비정상 JSON, 빈 입력
- 이벤트 기록: `events` 테이블 insertEvent() 확인
- 통계 기록: `events` 테이블 `subagent_stop` 타입 insertEvent() 확인
- exit 코드: 모든 경우 0

### 통합 테스트

- 훅 등록 후 실제 SubagentStop 이벤트 처리 흐름

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] Phase 1부터 순차 구현 시작
3. [ ] 훅 등록 JSON을 settings.json에 추가
