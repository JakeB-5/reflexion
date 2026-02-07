---
spec: ai-analyzer
created: 2026-02-07
domain: ai-analysis
---

# 구현 계획: AI 분석 실행 모듈 (ai-analyzer)

## 개요

`claude --print`를 래핑하여 수집 로그를 AI로 분석하고 결과를 캐시하는 핵심 모듈. 프롬프트 템플릿 관리와 JSON 파싱을 포함한다.

---

## Phase 1: 기반 구축

### 1.1 프롬프트 템플릿 생성
- `~/.self-generation/prompts/analyze.md` 템플릿 파일 생성
- 플레이스홀더: `{{days}}`, `{{project}}`, `{{log_data}}`, `{{feedback_history}}`, `{{existing_skills}}`, `{{outcome_metrics}}`
- DESIGN.md 5.2절의 템플릿을 그대로 구현

### 1.2 모듈 스캐폴딩
- `~/.self-generation/lib/ai-analyzer.mjs` 파일 생성
- 상수 정의: `GLOBAL_DIR`, `CACHE_FILE`, `PROMPT_TEMPLATE`
- 의존 모듈 import: `log-writer.mjs`, `feedback-tracker.mjs`, `skill-matcher.mjs`

---

## Phase 2: 핵심 로직 구현

### 2.1 summarizeForPrompt()
- 로그 엔트리를 프롬프트 크기에 적합하게 요약
- 프롬프트 최대 100개 제한, 세션별 도구 시퀀스 집계
- 에러 축약 형태 생성

### 2.2 buildPrompt()
- `prompts/analyze.md` 로드 및 플레이스홀더 치환
- 피드백 이력 (feedback-tracker), 기존 스킬 (skill-matcher), 효과 메트릭 주입

### 2.3 extractJSON()
- ````json` 코드 블록 내 JSON 추출
- 순수 JSON 패턴 폴백
- 정규식 기반 추출

### 2.4 runAnalysis()
- `execSync`로 `claude --print` 실행
- 최소 5개 프롬프트 검증
- JSON 파싱 및 `analysis-cache.json` 저장
- 에러 시 안전한 빈 결과 반환

### 2.5 runAnalysisAsync()
- `spawn`으로 `bin/analyze.mjs` 백그라운드 실행
- `detached: true`, `stdio: 'ignore'`, `child.unref()`

### 2.6 getCachedAnalysis()
- `analysis-cache.json` 읽기
- TTL 기반 만료 검증
- 파일 없음/파싱 실패 시 `null` 반환

---

## Phase 3: 테스트 및 검증

### 3.1 단위 테스트
- `summarizeForPrompt()`: 대량 엔트리 요약 검증, 최대 100개 제한 검증
- `buildPrompt()`: 플레이스홀더 치환 검증, 피드백 없음 케이스
- `extractJSON()`: 코드 블록, 순수 JSON, 잘못된 형식 케이스
- `getCachedAnalysis()`: 유효 캐시, 만료 캐시, 파일 없음 케이스

### 3.2 통합 테스트
- `runAnalysis()`: mock `claude --print`로 전체 플로우 검증
- `runAnalysisAsync()`: 자식 프로세스 생성 검증

---

## 의존성

- `data-collection/log-writer`: `getLogFile()`, `readEntries()` 함수
- `suggestion-engine/feedback-tracker` (Phase 4 이후): `getFeedbackSummary()`
- `realtime-assist/skill-matcher` (Phase 5 이후): `loadSkills()`
