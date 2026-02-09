---
name: ai-analysis
description: "Phase 2: AI 기반 패턴 분석 레이어"
created: 2026-02-07
status: active
depends_on:
  - data-collection
---

# Domain: ai-analysis

> Phase 2: AI 기반 패턴 분석 레이어 — claude --print로 수집 데이터를 의미 분석하여 패턴 감지 및 제안 생성

## 범위

### 유틸리티 (lib/)
- `ai-analyzer.mjs` — claude --print 실행, 동기/비동기, 캐시 관리, 로그 요약

### 프롬프트 템플릿 (prompts/)
- `analyze.md` — AI 분석 프롬프트 (클러스터, 워크플로우, 에러패턴, 제안, 시노님맵)

### CLI (bin/)
- `analyze.mjs` — 수동 AI 분석 실행 도구

### 훅 확장
- `session-summary.mjs` 확장 — SessionEnd에서 AI 분석 비동기 트리거
- `session-analyzer.mjs` — SessionStart에서 캐시된 분석 결과 주입

### 데이터 (SQLite 테이블)
- `analysis_cache` 테이블 — AI 분석 결과 캐시 (content-addressable, TTL 기반)

## 핵심 원칙
- AI 분석은 SessionEnd에서 비동기 실행 (세션 블로킹 없음)
- SessionStart에서는 캐시만 읽기 (<100ms)
- 프롬프트 5개 미만이면 분석 생략
- 로그를 요약하여 토큰 절약 (summarizeForPrompt)
- 피드백 이력 + 기존 스킬 + 효과 메트릭을 프롬프트에 주입

## DESIGN.md 참조 섹션
- 5.1 분석 전략
- 5.2 AI 분석 프롬프트 템플릿
- 5.3 AI 분석 실행 모듈
- 5.4 SessionEnd 훅 (AI 분석 트리거)
- 5.5 SessionStart 훅 (캐시 주입)
- 5.6 심층 분석 CLI

## 연결 스펙

| 스펙 ID | 제목 | 의존성 |
|---------|------|--------|
| ai-analyzer | AI 분석 실행 모듈 | data-collection/log-writer |
| analyze-cli | 심층 분석 CLI 도구 | ai-analysis/ai-analyzer |
| session-start-hook | SessionStart 캐시 주입 훅 | ai-analysis/ai-analyzer, data-collection/log-writer |
