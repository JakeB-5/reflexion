---
feature: skill-matcher
created: 2026-02-07
status: draft
---

# 구현 계획: skill-matcher

> `lib/skill-matcher.mjs` — 프롬프트-스킬 매칭 유틸리티 모듈 구현 계획

---

## 개요

skill-matcher는 사용자 프롬프트를 기존 커스텀 스킬과 매칭하는 하이브리드 시스템이다. AI 시노님맵(의미 매칭)과 키워드 패턴 매칭(구문 매칭)을 결합하여 관련 스킬을 자동 감지한다.

---

## 기술 결정

### 결정 1: 시노님맵 우선 매칭 순서

**근거:** AI가 생성한 시노님맵은 의미적 유사도가 높아 더 정확하다. 키워드 매칭은 fallback으로 사용한다.

**대안 검토:**
- 점수 합산 방식: 구현 복잡도 증가, 실시간 훅 환경에 부적합
- 키워드 우선: AI 분석 결과 활용도 감소

### 결정 2: 50% 키워드 임계값

**근거:** DESIGN.md 명세에 따른 값. 너무 낮으면 오탐이 많고, 너무 높으면 유용한 매칭을 놓친다. 실사용 후 피드백 기반으로 조정 가능하다.

---

## 구현 단계

### Phase 1: 스캐폴드 및 스킬 로드

모듈 파일 생성, `loadSkills()` 및 `loadSynonymMap()` 구현

**산출물:**
- [ ] `lib/skill-matcher.mjs` 파일 생성 (CACHE_FILE 상수, import)
- [ ] `loadSkills(projectPath)` 구현 — 전역/프로젝트 스킬 디렉토리 스캔
- [ ] `loadSynonymMap()` 구현 — analysis-cache.json에서 synonym_map 로드
- [ ] 단위 테스트: 디렉토리 부재, 빈 디렉토리, 정상 로드

### Phase 2: 핵심 매칭 로직

`extractPatterns()`과 `matchSkill()` 구현

**산출물:**
- [ ] `extractPatterns(content)` 구현 — "감지된 패턴" 섹션 파싱
- [ ] `matchSkill(prompt, skills, synonymMap)` 구현 — 시노님 → 키워드 매칭
- [ ] 단위 테스트: 시노님 매치, 키워드 매치, 임계값 미달, null 반환

### Phase 3: 통합 테스트

prompt-logger.mjs 훅 확장과의 통합 테스트

**산출물:**
- [ ] prompt-logger.mjs에서 `loadSkills()` + `matchSkill()` 호출 흐름 테스트
- [ ] 스킬 파일이 없는 환경에서의 graceful degradation 테스트
- [ ] 대소문자 무관 매칭 테스트

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| 시노님맵이 아직 생성되지 않은 초기 상태 | 🟢 LOW | 키워드 매칭 fallback으로 동작 |
| 스킬 파일에 "감지된 패턴" 섹션 없음 | 🟢 LOW | extractPatterns()가 빈 배열 반환, 매칭 스킵 |
| 다수의 스킬이 동시 매칭 | 🟡 MEDIUM | 첫 매칭 반환 (시노님 우선); 추후 점수 기반 순위 도입 가능 |

---

## 테스트 전략

### 단위 테스트

- `loadSkills()`: 전역+프로젝트 스킬 로드, 디렉토리 부재 처리
- `loadSynonymMap()`: 정상 로드, 파일 부재, 잘못된 JSON
- `extractPatterns()`: 패턴 섹션 있음/없음, 빈 파일
- `matchSkill()`: 시노님 매치, 키워드 50%+, 임계값 미달

### 통합 테스트

- prompt-logger.mjs 훅에서 전체 매칭 파이프라인 실행

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] Phase 1부터 순차 구현 시작
3. [ ] prompt-logger.mjs 훅 확장 (data-collection 도메인과 조율)
