---
name: data-collection
description: "Phase 1: 데이터 수집 레이어"
created: 2026-02-07
status: active
depends_on: []
---

# Domain: data-collection

> Phase 1: 데이터 수집 레이어 — 훅 스크립트를 통해 프롬프트, 도구 사용, 에러, 세션 요약을 SQLite DB에 기록

## 범위

### 훅 스크립트 (hooks/)
- `prompt-logger.mjs` — UserPromptSubmit: 프롬프트 수집
- `tool-logger.mjs` — PostToolUse: 도구 사용 기록 + resolution detection
- `error-logger.mjs` — PostToolUseFailure: 에러 수집 + 정규화
- `session-summary.mjs` — SessionEnd: 세션 요약 집계 (기본 버전)

### 유틸리티 (lib/)
- `db.mjs` — SQLite DB 연결, CRUD, 벡터 검색 유틸리티, WAL 모드 관리

### 데이터 스키마
- `prompt` 엔트리 (PromptEntry)
- `tool_use` 엔트리 (ToolUseEntry)
- `tool_error` 엔트리 (ToolErrorEntry)
- `session_summary` 엔트리 (SessionSummaryEntry)

### 설정
- `config.json` — 시스템 설정 (enabled, retentionDays, maxLogSizeBytes 등)

## 핵심 원칙
- 모든 훅은 non-blocking (exit 0, 빠른 완료)
- 전역 우선, 프로젝트 필터링 (project_path 기반 SQL 쿼리)
- Bash 명령어 첫 단어만 저장 (프라이버시)
- 스키마 버전 관리 (v: 1)
- SQLite WAL 모드로 훅 간 동시 DB 접근 지원

## DESIGN.md 참조 섹션
- 4.1 훅 등록
- 4.2 공통 유틸: SQLite DB 연결 및 CRUD
- 4.3 프롬프트 수집 훅
- 4.4 도구 사용 수집 훅
- 4.5 에러 수집 훅
- 4.6 세션 요약 훅
- 9.1~9.4 데이터 스키마

## 연결 스펙

| 스펙 | 파일 | REQ 범위 | 의존성 |
|------|------|----------|--------|
| log-writer | `specs/data-collection/log-writer/spec.md` | REQ-DC-001~012 | 없음 |
| prompt-logger | `specs/data-collection/prompt-logger/spec.md` | REQ-DC-101~104 | log-writer |
| tool-logger | `specs/data-collection/tool-logger/spec.md` | REQ-DC-201~205 | log-writer, realtime-assist/error-kb |
| error-logger | `specs/data-collection/error-logger/spec.md` | REQ-DC-301~304 | log-writer, realtime-assist/error-kb |
| session-summary | `specs/data-collection/session-summary/spec.md` | REQ-DC-401~404 | log-writer, ai-analysis/ai-analyzer |
