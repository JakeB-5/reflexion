---
name: suggestion-engine
description: "Phase 3+4: 제안 적용 엔진 + 피드백 루프"
created: 2026-02-07
status: active
depends_on:
  - ai-analysis
---

# Domain: suggestion-engine

> Phase 3+4: 제안 적용 엔진 + 피드백 루프 — AI가 생성한 제안을 적용/거부하고, 피드백을 추적하여 분석 품질을 개선

## 범위

### CLI (bin/)
- `apply.mjs` — 제안 적용 도구 (skill/claude_md/hook 3가지 유형)
- `dismiss.mjs` — 제안 거부 기록 도구

### 유틸리티 (lib/)
- `feedback-tracker.mjs` — 피드백 기록, 요약 조회, 스킬 사용률, 규칙 효과성, 미사용 스킬 탐지

### 제안 적용 대상 (3가지 출력 유형)
- **Custom Skills**: `.claude/commands/*.md` 생성 (전역/프로젝트)
- **CLAUDE.md Directives**: CLAUDE.md에 규칙 추가 (전역/프로젝트)
- **Hook Workflows**: `hooks/auto/workflow-*.mjs` 생성 + settings.json 등록

### 데이터
- `feedback.jsonl` — 제안 채택/거부 기록 (FeedbackEntry)

## 핵심 원칙
- 사용자 승인 없이 자동 적용 금지
- 피드백 데이터를 AI 프롬프트 컨텍스트로 주입 (정적 임계값 대신)
- 스킬 사용률, 규칙 효과성, 미사용 스킬 추적 (v7 P5)
- 훅 워크플로우는 반자동 (코드 생성 → 수동 등록 or --apply)

## DESIGN.md 참조 섹션
- 6.1 제안 적용 도구
- 6.1.1 제안 거부 도구
- 6.2 사용자 승인 플로우
- 7.1 채택 추적
- 7.2 피드백 → AI 프롬프트 주입

## 연결 스펙
- [suggestion-cli](../specs/suggestion-engine/suggestion-cli/spec.md) — 제안 적용/거부 CLI 도구
- [feedback-tracker](../specs/suggestion-engine/feedback-tracker/spec.md) — 피드백 추적 및 효과성 메트릭
