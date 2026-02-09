---
id: ai-analysis
created: 2026-02-07
---

# 도메인: ai-analysis

> Phase 2: AI 기반 패턴 분석 레이어

---

## 개요

### 범위 및 책임

이 도메인은 다음을 담당합니다:

- `claude --print`를 통한 수집 데이터의 의미 분석
- 패턴 감지 및 제안 생성 (스킬, CLAUDE.md 지침, 훅 워크플로우)
- 분석 결과 캐싱 및 SessionStart 시 주입

### 경계 정의

- **포함**: AI 분석 실행(`ai-analyzer.mjs`), 분석 프롬프트 템플릿, 분석 CLI, SessionStart 캐시 주입
- **제외**: 데이터 수집(data-collection), 제안 적용(suggestion-engine), 실시간 검색(realtime-assist)

---

## 의존성

- `data-collection` — 이벤트 데이터 조회(`db.mjs`)

---

## 스펙 목록

이 도메인에 속한 기능 명세:

| 스펙 ID | 상태 | 설명 |
|---------|------|------|
| ai-analyzer | draft | claude --print 실행, 캐시 관리, 로그 요약 모듈 |
| analyze-cli | draft | 수동 AI 분석 실행 CLI 도구 |
| session-start-hook | draft | SessionStart 캐시 주입 훅 |

---

## 공개 인터페이스

다른 도메인에서 사용할 수 있는 공개 API/인터페이스:

### 제공하는 기능

- `runAnalysis(options)` — AI 분석 실행 및 결과 반환
- `getCachedAnalysis(projectPath)` — 캐시된 분석 결과 조회
- `summarizeForPrompt(events)` — 이벤트 로그를 AI 프롬프트용으로 요약

### 제공하는 타입/인터페이스

- `AnalysisResult` — AI 분석 결과 스키마 (suggestions, synonymMap, patterns)
- `AnalysisCacheEntry` — 분석 캐시 엔트리 스키마 (content-addressable)

---

## 비고

AI 분석은 SessionEnd에서 비동기 실행되며, SessionStart에서는 캐시만 읽는다(<100ms). 프롬프트 5개 미만이면 분석을 생략한다.
