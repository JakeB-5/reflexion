---
name: realtime-assist
description: "Phase 5: 실시간 어시스턴스 레이어"
created: 2026-02-07
status: active
depends_on:
  - data-collection
  - ai-analysis
---

# Domain: realtime-assist

> Phase 5: 실시간 어시스턴스 레이어 — 세션 내 즉시 도움 제공 (에러 KB, 스킬 매칭, 서브에이전트 추적, 사전 예방 가이드)

## 범위

### 유틸리티 (lib/)
- `error-kb.mjs` — 에러 해결 이력 KB (검색 + 기록, substring match 포함)
- `skill-matcher.mjs` — 스킬-프롬프트 매칭 (시노님맵 + 키워드 매칭)

### 훅 스크립트 (hooks/)
- `subagent-tracker.mjs` — SubagentStop: 서브에이전트 성능 추적
- `pre-tool-guide.mjs` — PreToolUse: 사전 예방 가이드 (v7 P1)
- `subagent-context.mjs` — SubagentStart: 서브에이전트에 학습 데이터 주입 (v7 P9)

### 훅 확장 (기존 훅에 실시간 기능 추가)
- `error-logger.mjs` 확장 — PostToolUseFailure에서 에러 KB 실시간 검색
- `prompt-logger.mjs` 확장 — UserPromptSubmit에서 스킬 자동 감지 + 사용 추적
- `tool-logger.mjs` 확장 — PostToolUse에서 resolution detection (v7: 크로스도구)
- `session-analyzer.mjs` 확장 — SessionStart에서 이전 세션 컨텍스트 주입

### 데이터
- `error-kb.jsonl` — 에러 해결 이력 (ErrorKBEntry)
- `subagent-stats.jsonl` — 서브에이전트 성능 메트릭
- `skill_used` 엔트리 (SkillUsedEntry) — prompt-log.jsonl 내

## 핵심 원칙
- 배치 분석(Phase 2)과 상호 보완: 배치는 장기 패턴, 실시간은 즉각 대응
- 에러 KB는 정규화된 에러로 검색 + substring fallback
- 스킬 매칭은 AI 시노님맵(배치) + 키워드 매칭(실시간) 하이브리드
- 서브에이전트 컨텍스트는 코드 작업 에이전트에만 주입 (500자 제한)
- PreToolUse 가이드는 Edit/Write, Bash, Task 도구에만 적용

## DESIGN.md 참조 섹션
- 8.1 에러 KB 실시간 검색
- 8.2 스킬 자동 감지
- 8.3 서브에이전트 성능 추적
- 8.4 세션 간 컨텍스트 주입
- 8.5 사전 예방 가이드 훅 (P1, v7)
- 8.6 서브에이전트 컨텍스트 주입 훅 (P9, v7)
- 8.7 훅 등록 (v6+v7)

## 연결 스펙
(스펙 작성 후 연결)
