---
id: infra
created: 2026-02-09
---

# 도메인: infra

> 인프라: 설치, 설정, 시스템 관리

---

## 개요

### 범위 및 책임

이 도메인은 다음을 담당합니다:

- 시스템 설치 및 제거 CLI (`bin/install.mjs`)
- 설정 스키마 정의 및 검증 (`config.json`)
- Claude Code Hooks API 훅 등록/해제 관리

### 경계 정의

- **포함**: 설치/제거 CLI, 설정 스키마, 훅 등록 관리
- **제외**: 데이터 수집(data-collection), AI 분석(ai-analysis), 제안 엔진(suggestion-engine), 실시간 어시스턴스(realtime-assist)

---

## 의존성

이 도메인은 다른 도메인에 의존하지 않습니다. (인프라 기반 레이어)

---

## 스펙 목록

이 도메인에 속한 기능 명세:

| 스펙 ID | 상태 | 설명 |
|---------|------|------|
| install-cli | draft | 시스템 설치/제거 CLI 스크립트 |
| config-schema | draft | 설정 스키마 정의 및 검증 |

---

## 공개 인터페이스

다른 도메인에서 사용할 수 있는 공개 API/인터페이스:

### 제공하는 기능

- `install.mjs` — 시스템 설치 CLI (디렉토리 생성, 의존성 설치, 훅 등록)
- `install.mjs --uninstall` — 시스템 제거 CLI (훅 해제, 선택적 데이터 삭제)

### 제공하는 설정

- `config.json` — 시스템 전체 설정 (enabled, retentionDays, embedding 등)

---

## 비고

이 도메인은 Phase 1 로드맵의 작업 0번으로, 모든 다른 도메인보다 먼저 실행되어야 하는 전제조건이다. 설치 CLI가 디렉토리 구조, 의존성, 훅 등록을 완료해야 다른 훅 스크립트가 정상 동작한다.
