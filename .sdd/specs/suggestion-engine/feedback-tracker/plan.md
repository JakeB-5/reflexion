---
feature: feedback-tracker
created: 2026-02-07
status: draft
---

# 구현 계획: feedback-tracker

> 제안 채택/거부 피드백을 기록하고, 요약 및 효과성 메트릭을 산출하는 유틸리티 모듈

---

## 개요

`lib/feedback-tracker.mjs`에 5개 함수를 구현한다. 핵심은 `recordFeedback()`(기록)과 `getFeedbackSummary()`(요약)이며, 보조 메트릭 함수 3개(`calcSkillUsageRate`, `calcRuleEffectiveness`, `findStaleSkills`)가 요약에 포함된다.

---

## 기술 결정

### 결정 1: append-only JSONL 저장

**근거:** 피드백 데이터의 불변성 보장. 수정/삭제 없이 추가만 허용하여 데이터 손실 방지. JSONL은 라인 단위 파싱으로 대용량 파일에서도 메모리 효율적.

**대안 검토:**
- JSON 배열 — 전체 읽기/쓰기 필요, 동시성 문제
- SQLite — 외부 의존성 금지 제약에 위배

### 결정 2: 에러 시 null/[] 반환 (예외 미전파)

**근거:** 메트릭 함수는 보조 정보 제공 목적. 파일 부재나 파싱 실패가 상위 로직을 중단시키면 안 됨. AI 분석 프롬프트에 `null`이 주입되면 해당 섹션을 건너뜀.

**대안 검토:**
- 예외 throw — 호출자가 모두 try-catch 필요, 과도
- 기본값 객체 반환 — null과 기본값 구분이 모호

---

## 구현 단계

### Phase 1: 스캐폴딩

모듈 기본 구조 및 상수 정의. `DATA_DIR`, `FEEDBACK_FILE` 경로 설정.

**산출물:**
- [ ] `lib/feedback-tracker.mjs` 모듈 구조 (import, 상수, export)
- [ ] `feedback.jsonl` 스키마 정의 (`{ v, ts, suggestionId, action, ...details }`)

### Phase 2: 핵심 함수

5개 함수 순차 구현. `recordFeedback` → `getFeedbackSummary` → 보조 메트릭 3개.

**산출물:**
- [ ] `recordFeedback(id, action, details)` — appendFileSync로 JSONL 기록
- [ ] `getFeedbackSummary()` — 전체 파싱, 채택/거부 집계, 최근 10건 슬라이스
- [ ] `calcSkillUsageRate()` — prompt-log.jsonl에서 skill_used / skill_created 비율
- [ ] `calcRuleEffectiveness()` — tool_error 전체/최근 7일 집계
- [ ] `findStaleSkills(days)` — 스킬 목록 대비 사용 이력 대조

### Phase 3: 테스트 및 검증

각 함수별 정상/경계/에러 케이스 테스트.

**산출물:**
- [ ] `recordFeedback` 테스트 (채택/거부/무시 기록, 스키마 검증)
- [ ] `getFeedbackSummary` 테스트 (정상, 파일 없음, 최근 이력 제한)
- [ ] 보조 메트릭 함수 테스트 (정상 계산, 데이터 없음, 파일 없음)

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| prompt-log.jsonl 대용량 시 파싱 지연 | MEDIUM | 필요 시 라인 스트리밍 도입 (Phase 2에서는 전체 읽기) |
| JSONL 깨진 라인 | LOW | try-catch로 개별 라인 파싱, 실패 시 skip |
| loadSkills() 함수 미정의 | MEDIUM | findStaleSkills에서 스킬 목록 로드 방법 확정 필요 |

---

## 테스트 전략

### 단위 테스트

- 각 함수별 독립 테스트 (임시 디렉터리에 테스트 파일 생성)
- 경계 케이스: 빈 파일, 깨진 JSONL 라인, 파일 부재
- 커버리지 목표: 80% 이상

### 통합 테스트

- `recordFeedback` → `getFeedbackSummary` 연쇄 호출 검증
- suggestion-cli의 apply/dismiss와 연동 테스트

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] log-writer 모듈의 JSONL 읽기 인터페이스 확인 (의존성)
3. [ ] `loadSkills()` 함수의 스킬 목록 로드 방법 확정
4. [ ] 구현 시작
