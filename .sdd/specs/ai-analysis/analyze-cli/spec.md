---
id: analyze-cli
title: "심층 분석 CLI 도구"
status: draft
created: 2026-02-07
domain: ai-analysis
depends: "ai-analysis/ai-analyzer, data-collection/log-writer"
constitution_version: "2.0.0"
---

# 심층 분석 CLI 도구 (analyze-cli)

> 사용자가 수동으로 AI 패턴 분석을 실행할 수 있는 CLI 도구 (`bin/analyze.mjs`). `--days`와 `--project` 플래그를 지원하며, 분석 결과를 콘솔에 포맷팅하여 출력한다.

---

## 요구사항

### REQ-AA-101: CLI 인자 파싱

시스템은 `--days`와 `--project` CLI 인자를 파싱해야 한다(SHALL). `--days`의 기본값은 30이어야 하며(SHALL), `--project`가 생략되면 전역 분석(project=null)을 수행해야 한다(SHALL).

#### Scenario: 기본 인자로 실행

- **GIVEN** 사용자가 `node ~/.self-generation/bin/analyze.mjs`를 인자 없이 실행한다
- **WHEN** CLI가 인자를 파싱한다
- **THEN** `days=30`, `project=null`로 `runAnalysis()`가 호출된다

#### Scenario: 커스텀 인자로 실행

- **GIVEN** 사용자가 `node ~/.self-generation/bin/analyze.mjs --days 14 --project my-app`을 실행한다
- **WHEN** CLI가 인자를 파싱한다
- **THEN** `days=14`, `project='my-app'`으로 `runAnalysis()`가 호출된다

---

### REQ-AA-102: 데이터 부족 처리

시스템은 분석 대상 프롬프트가 5개 미만일 때 사용자에게 안내 메시지를 출력하고 정상 종료해야 한다(SHALL).

#### Scenario: 프롬프트 부족 시 안내 메시지

- **GIVEN** `events` 테이블에 최근 30일간 프롬프트가 3개만 기록되어 있다
- **WHEN** `runAnalysis()`가 `{ suggestions: [], reason: 'insufficient_data' }`를 반환한다
- **THEN** `'데이터 부족: 프롬프트 5개 이상 수집 후 다시 실행하세요.'`가 콘솔에 출력되고 exit code 0으로 종료된다

---

### REQ-AA-103: 분석 결과 포맷팅 출력

시스템은 AI 분석 결과를 클러스터, 워크플로우, 에러 패턴, 제안의 4개 섹션으로 구분하여 콘솔에 출력해야 한다(SHALL). 각 섹션은 데이터가 존재할 때만 출력해야 한다(SHALL).

#### Scenario: 전체 결과 출력

- **GIVEN** AI 분석이 clusters 3개, workflows 2개, errorPatterns 1개, suggestions 4개를 반환했다
- **WHEN** 결과를 콘솔에 출력한다
- **THEN** 다음 4개 섹션이 순서대로 출력된다: `'--- 반복 프롬프트 클러스터 ---'` (각 클러스터의 count, intent, summary, 예시 최대 3개), `'--- 반복 도구 시퀀스 ---'` (각 워크플로우의 count, pattern, purpose), `'--- 반복 에러 패턴 ---'` (각 에러의 count, pattern, proposedRule), `'=== 개선 제안 ==='` (번호, type, summary, evidence, action)

#### Scenario: 일부 섹션만 존재

- **GIVEN** AI 분석이 clusters 2개와 suggestions 1개만 반환하고, workflows와 errorPatterns는 빈 배열이다
- **WHEN** 결과를 콘솔에 출력한다
- **THEN** 클러스터와 제안 섹션만 출력되고, 워크플로우와 에러 패턴 섹션은 출력되지 않는다

---

### REQ-AA-104: 분석 실패 처리

시스템은 `claude --print` 실행 실패 시 사용자에게 에러 메시지를 출력하고 exit code 1로 종료해야 한다(SHALL).

#### Scenario: AI 분석 에러 시 종료

- **GIVEN** `claude` CLI가 설치되지 않았거나 네트워크 오류가 발생했다
- **WHEN** `runAnalysis()`가 `{ suggestions: [], error: 'Command failed: claude --print' }`를 반환한다
- **THEN** `'분석 실패: Command failed: claude --print'`가 stderr에 출력되고 exit code 1로 종료된다

---

## 비기능 요구사항

### 사용성

- CLI 실행 시 분석 기간과 대상을 나타내는 헤더를 출력해야 한다(SHALL): `'=== Self-Generation AI 패턴 분석 (최근 {days}일) ==='`
- 제안 적용 안내를 마지막에 출력해야 한다(SHALL): `'제안을 적용하려면: node ~/.self-generation/bin/apply.mjs <번호>'`

### 성능

- CLI 자체의 오버헤드(인자 파싱, 포맷팅)는 100ms 이내여야 한다(SHOULD)
- 전체 실행 시간은 `runAnalysis()` 소요 시간에 의존한다

---

## 제약사항

- 외부 인자 파싱 라이브러리 없이 `process.argv`를 직접 파싱해야 한다(SHALL)
- `ai-analysis/ai-analyzer` 모듈의 `runAnalysis()`에 의존한다
- 데이터 접근은 `lib/db.mjs`의 `queryEvents()` 등을 통해 SQLite DB(`self-gen.db`)에서 수행한다
- Node.js >= 18 ES Modules 환경에서 실행된다

---

## 용어 정의

| 용어 | 정의 |
|------|------|
| `bin/analyze.mjs` | 수동 AI 분석을 실행하는 CLI 도구 엔트리포인트 |
| `--days` | 분석 대상 기간 (일 단위, 기본값 30) |
| `--project` | 분석 대상 프로젝트 필터 (생략 시 전역 분석) |
