---
name: infra
description: "인프라: 설치, 설정, 시스템 관리"
created: 2026-02-09
status: active
depends_on: []
---

# Domain: infra

> 인프라 레이어 — 시스템 설치/제거, 설정 스키마 관리, 훅 등록 등 cross-cutting 인프라 기능

## 범위

### CLI 도구 (bin/)
- `install.mjs` — 시스템 설치/제거 CLI (디렉토리, 의존성, 훅 등록)

### 설정
- `config.json` — 시스템 전체 설정 스키마 정의 및 검증

## 핵심 원칙
- 멱등성 보장 (반복 실행 시 동일 결과)
- 안전한 제거 (다른 훅/설정에 영향 없음)
- 최소 의존성 (Node.js 내장 모듈 우선)
- 사용자 확인 없이 기존 데이터 삭제 금지

## DESIGN.md 참조 섹션
- 6.1.2 설치 스크립트
- 9.6 config.json
- 11. 구현 로드맵 (Phase 1, 작업 0)

## 연결 스펙

| 스펙 | 파일 | REQ 범위 | 의존성 |
|------|------|----------|--------|
| install-cli | `specs/infra/install-cli/spec.md` | REQ-INF-001~007 | 없음 |
| config-schema | `specs/infra/config-schema/spec.md` | REQ-INF-101~1xx | 없음 |
