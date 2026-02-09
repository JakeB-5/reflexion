---
feature: log-writer
created: 2026-02-07
total: 10
completed: 0
---

# 작업 목록: log-writer

## 개요

- 총 작업 수: 10개
- 예상 복잡도: 중간

---

## 작업 목록

### Phase 1: 기반 구축

- [ ] [P1] `lib/db.mjs` 파일 생성 및 ES Module 설정
- [ ] [P1] `getDb()` 구현 — SQLite DB 연결 반환 + `data/` 디렉토리 자동 생성
- [ ] [P1] `getProjectName(cwd)` 구현 — cwd에서 프로젝트 디렉토리명 추출
- [ ] [P1] `readStdin()` 구현 — stdin 동기 읽기 + JSON 파싱 (64KB 버퍼 반복)

### Phase 2: 핵심 구현

- [ ] [P2] `insertEvent(entry)` 구현 — `events` 테이블에 이벤트 INSERT
- [ ] [P2] `queryEvents(filterOrLimit)` 구현 — 필터 조회(type/sessionId/project/projectPath/since) + 숫자 축약 조회
- [ ] [P2] `pruneOldEvents()` 구현 — 확률적 정리 (1% 확률 트리거)
- [ ] [P2] `loadConfig()` 구현 — config.json 로딩 + 7개 기본값 fallback

### Phase 3: 마무리

- [ ] [P3] [->T] 단위 테스트 작성 — insertEvent, queryEvents, pruneOldEvents, loadConfig, readStdin 각 함수별

---

## 의존성 그래프

```mermaid
graph LR
    A[파일 생성] --> B[getDb]
    A --> C[getProjectName]
    A --> D[readStdin]
    B --> E[insertEvent]
    B --> F[queryEvents]
    E --> G[pruneOldEvents]
    B --> I[loadConfig]
    E & F & G & I --> J[단위 테스트]
```

---

## 마커 범례

| 마커 | 의미 |
|------|------|
| [P1-3] | 우선순위 |
| [->T] | 테스트 필요 |
| [US] | 불확실/검토 필요 |
