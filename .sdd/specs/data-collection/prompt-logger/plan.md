---
feature: prompt-logger
created: 2026-02-07
status: draft
---

# 구현 계획: prompt-logger

> UserPromptSubmit 훅 — 프롬프트 수집 및 스킬 사용 이벤트 기록

---

## 개요

`hooks/prompt-logger.mjs`는 사용자 프롬프트 입력 시 트리거되는 훅 스크립트이다. 프롬프트 원문, 프로젝트 정보, 문자 수를 기록하고, 슬래시 커맨드 감지 시 별도 스킬 사용 이벤트를 기록한다.

---

## 기술 결정

### 결정 1: 수집 시점에 키워드/인텐트 분석 제외

**근거:** v5 설계 변경에 따라, 수집 훅은 원본 데이터만 빠르게 기록하고 분석은 AI 배치 단계에서 수행

**대안 검토:**
- 수집 시 키워드 추출 — 훅 실행 시간 증가, 정적 분석 품질 한계
- 수집 시 인텐트 분류 — AI 호출 필요하여 non-blocking 원칙 위반

### 결정 2: 프라이버시 모드 지원

**근거:** `collectPromptText: false` 설정으로 프롬프트 원문 수집 비활성화. GDPR 등 규정 준수 선택지 제공

---

## 구현 단계

### Phase 1: 스캐폴드

훅 스크립트 파일 생성 및 log-writer import 설정

**산출물:**
- [ ] `~/.self-generation/hooks/prompt-logger.mjs` 파일 생성
- [ ] log-writer.mjs import 및 ES Module 설정

### Phase 2: 핵심 로직

프롬프트 수집, 프라이버시 모드, 스킬 감지 구현

**산출물:**
- [ ] PromptEntry 스키마로 프롬프트 기록
- [ ] `collectPromptText` 설정에 따른 텍스트 수집 제어
- [ ] 슬래시 커맨드 감지 및 `skill_used` 이벤트 기록
- [ ] try-catch + exit(0) non-blocking 래퍼

### Phase 3: 테스트

시나리오 기반 테스트

**산출물:**
- [ ] 일반 프롬프트 기록 검증
- [ ] 프라이버시 모드 (`[REDACTED]`) 검증
- [ ] 슬래시 커맨드 감지 (`/ts-init` → `skill_used`) 검증
- [ ] 잘못된 stdin 처리 검증

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| stdin JSON 파싱 실패 | LOW | try-catch로 exit(0) 보장 |
| config.json 부재 | LOW | loadConfig 기본값 활용 |

---

## 테스트 전략

### 단위 테스트

- PromptEntry 스키마 검증
- 슬래시 커맨드 파싱 로직
- 커버리지 목표: 80% 이상

### 통합 테스트

- stdin mock으로 전체 훅 실행 플로우 검증
- prompt-log.jsonl 파일 기록 결과 확인

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] 작업 분해 (tasks.md)
3. [ ] 구현 시작
