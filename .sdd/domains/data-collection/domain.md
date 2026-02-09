---
id: data-collection
created: 2026-02-07
---

# 도메인: data-collection

> Phase 1: 데이터 수집 레이어

---

## 개요

### 범위 및 책임

이 도메인은 다음을 담당합니다:

- 훅 스크립트를 통한 프롬프트, 도구 사용, 에러, 세션 요약 수집
- SQLite DB 연결 및 CRUD 유틸리티 제공
- 이벤트 데이터의 스키마 정의 및 버전 관리

### 경계 정의

- **포함**: 훅 이벤트 수집(4개 훅), DB 유틸리티(`db.mjs`), 이벤트 스키마 정의
- **제외**: AI 분석(ai-analysis), 제안 적용(suggestion-engine), 실시간 검색(realtime-assist)

---

## 의존성

이 도메인은 다른 도메인에 의존하지 않습니다. (기반 레이어)

---

## 스펙 목록

이 도메인에 속한 기능 명세:

| 스펙 ID | 상태 | 설명 |
|---------|------|------|
| log-writer | draft | SQLite DB 연결, CRUD, 벡터 검색 유틸리티 |
| prompt-logger | draft | UserPromptSubmit 프롬프트 수집 훅 |
| tool-logger | draft | PostToolUse 도구 사용 기록 훅 |
| error-logger | draft | PostToolUseFailure 에러 수집 훅 |
| session-summary | draft | SessionEnd 세션 요약 집계 훅 |

---

## 공개 인터페이스

다른 도메인에서 사용할 수 있는 공개 API/인터페이스:

### 제공하는 기능

- `getDb()` — SQLite DB 연결 인스턴스 반환
- `insertEvent(type, data)` — events 테이블에 이벤트 INSERT
- `queryEvents(filters)` — 이벤트 조회 (project_path, type, 기간 필터)

### 제공하는 타입/인터페이스

- `PromptEntry` — 프롬프트 이벤트 스키마
- `ToolUseEntry` — 도구 사용 이벤트 스키마
- `ToolErrorEntry` — 에러 이벤트 스키마
- `SessionSummaryEntry` — 세션 요약 스키마

---

## 비고

모든 훅은 non-blocking으로 동작하며 exit 0으로 종료한다. SQLite WAL 모드로 훅 간 동시 DB 접근을 지원한다.
