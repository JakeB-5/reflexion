---
feature: error-logger
created: 2026-02-07
status: draft
---

# 구현 계획: error-logger

> PostToolUseFailure 훅 — 에러 수집, 정규화, KB 실시간 검색

---

## 개요

`hooks/error-logger.mjs`는 도구 실행 실패 시 트리거되는 훅 스크립트이다. 에러 메시지를 정규화하여 기록하고, 에러 KB에서 과거 해결 이력을 검색하여 Claude에게 `additionalContext`로 주입한다.

---

## 기술 결정

### 결정 1: 에러 정규화 전략 (PATH → N → STR)

**근거:** 동일 유형 에러의 패턴 매칭을 위해 가변 부분(경로, 숫자, 문자열)을 플레이스홀더로 치환. 치환 순서가 결과에 영향 — 경로를 먼저 처리하여 경로 내 숫자가 별도 치환되지 않도록 함

**대안 검토:**
- ML 기반 에러 클러스터링 — 정확하지만 의존성과 비용 과다
- 단순 해시 비교 — 유사 에러 매칭 불가

### 결정 2: stdout을 통한 additionalContext 주입

**근거:** Claude Code Hook API 공식 규격에 따라 stdout JSON으로 컨텍스트 주입

---

## 구현 단계

### Phase 1: 스캐폴드

훅 스크립트 파일 및 의존성 설정

**산출물:**
- [ ] `~/.self-generation/hooks/error-logger.mjs` 파일 생성
- [ ] log-writer.mjs import 설정

### Phase 2: 핵심 로직

에러 수집, 정규화, KB 검색, 컨텍스트 주입 구현

**산출물:**
- [ ] ToolErrorEntry 스키마로 에러 기록
- [ ] `normalizeError()` — 경로/숫자/문자열 정규화 + 200자 제한
- [ ] `searchErrorKB()` 통합 (Phase 5 활성화)
- [ ] stdout `additionalContext` JSON 출력
- [ ] try-catch + exit(0) non-blocking 래퍼

### Phase 3: 테스트

정규화 로직 및 KB 검색 테스트

**산출물:**
- [ ] normalizeError 각 치환 규칙별 테스트
- [ ] 치환 순서 의존성 테스트
- [ ] 200자 절단 테스트
- [ ] KB 매칭 시 additionalContext 출력 검증
- [ ] KB 미매칭 시 stdout 무출력 검증

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| 정규화 오탐 (과도한 치환) | MEDIUM | 100자 이내 문자열만 치환, 200자 제한 |
| error-kb.jsonl 손상 | LOW | try-catch로 KB 검색 실패 시 정상 종료 |
| stdout 출력 형식 오류 | MEDIUM | JSON.stringify로 직렬화 보장 |

---

## 테스트 전략

### 단위 테스트

- normalizeError 함수 독립 테스트 (다양한 에러 메시지 패턴)
- 커버리지 목표: 80% 이상

### 통합 테스트

- stdin mock으로 전체 훅 실행 플로우 검증
- additionalContext 출력 형식 검증

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] 작업 분해 (tasks.md)
3. [ ] 구현 시작
