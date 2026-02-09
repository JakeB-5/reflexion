---
id: suggestion-engine
created: 2026-02-07
---

# 도메인: suggestion-engine

> Phase 3+4: 제안 적용 엔진 + 피드백 루프

---

## 개요

### 범위 및 책임

이 도메인은 다음을 담당합니다:

- AI가 생성한 제안의 적용 및 거부 처리 (CLI 도구)
- 피드백 추적 (수락률, 스킬 사용률, 규칙 효과성)
- 피드백 데이터를 AI 프롬프트 컨텍스트로 주입하여 분석 품질 개선

### 경계 정의

- **포함**: 제안 적용 CLI(`apply.mjs`, `dismiss.mjs`), 피드백 추적(`feedback-tracker.mjs`), 3가지 출력 유형 생성
- **제외**: AI 분석(ai-analysis), 데이터 수집(data-collection), 실시간 검색(realtime-assist)

---

## 의존성

- `ai-analysis` — AI 분석 결과(제안) 소비

---

## 스펙 목록

이 도메인에 속한 기능 명세:

| 스펙 ID | 상태 | 설명 |
|---------|------|------|
| suggestion-cli | draft | 제안 적용/거부 CLI 도구 (apply.mjs, dismiss.mjs) |
| feedback-tracker | draft | 피드백 기록, 수락률 계산, 효과성 메트릭 |

---

## 공개 인터페이스

다른 도메인에서 사용할 수 있는 공개 API/인터페이스:

### 제공하는 기능

- `recordFeedback(suggestionId, action, reason)` — 제안 채택/거부 기록
- `getFeedbackSummary(projectPath)` — 피드백 요약 조회 (수락률, 효과성)
- `getSkillUsageRate()` — 스킬 사용률 조회
- `getUnusedSkills()` — 미사용 스킬 탐지

### 제공하는 타입/인터페이스

- `FeedbackEntry` — 피드백 기록 스키마
- `FeedbackSummary` — 피드백 요약 스키마

---

## 비고

사용자 승인 없이 자동 적용은 금지된다. 피드백 데이터는 정적 임계값 대신 AI 프롬프트 컨텍스트로 주입되어 분석 품질을 개선한다.
