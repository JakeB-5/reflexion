---
spec: analyze-cli
created: 2026-02-07
domain: ai-analysis
---

# 구현 계획: 심층 분석 CLI 도구 (analyze-cli)

## 개요

사용자가 수동으로 AI 패턴 분석을 실행하고 결과를 콘솔에서 확인할 수 있는 CLI 도구. `ai-analyzer` 모듈의 `runAnalysis()`를 호출하고 결과를 포맷팅한다.

---

## Phase 1: 기반 구축

### 1.1 CLI 엔트리포인트 생성
- `~/.self-generation/bin/analyze.mjs` 파일 생성
- `process.argv` 기반 인자 파싱 (`--days`, `--project`)
- 기본값 설정: `days=30`, `project=null`

---

## Phase 2: 핵심 구현

### 2.1 분석 실행 연동
- `ai-analyzer.mjs`의 `runAnalysis()` import 및 호출
- 분석 헤더 출력: `=== Self-Generation AI 패턴 분석 (최근 {days}일) ===`

### 2.2 결과 분기 처리
- `result.error` → stderr 출력 + exit 1
- `result.reason === 'insufficient_data'` → 안내 메시지 + exit 0
- 정상 결과 → 포맷팅 출력

### 2.3 결과 포맷팅
- 클러스터 섹션: count, intent, summary, 예시 최대 3개
- 워크플로우 섹션: count, pattern, purpose
- 에러 패턴 섹션: count, pattern, proposedRule
- 제안 섹션: 번호, type, summary, evidence, action
- 적용 안내 푸터: `제안을 적용하려면: node ~/.self-generation/bin/apply.mjs <번호>`

---

## Phase 3: 테스트 및 검증

### 3.1 단위 테스트
- 인자 파싱: 기본값, 커스텀 값, 잘못된 입력
- 결과 분기: error, insufficient_data, 정상 결과
- 포맷팅: 각 섹션 존재/부재 조합

### 3.2 수동 검증
- 실제 `claude --print` 연동 E2E 테스트

---

## 의존성

- `ai-analysis/ai-analyzer`: `runAnalysis()` 함수
