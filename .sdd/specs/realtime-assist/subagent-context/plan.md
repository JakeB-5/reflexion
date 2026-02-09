---
feature: subagent-context
created: 2026-02-07
status: draft
---

# 구현 계획: subagent-context

> `hooks/subagent-context.mjs` — SubagentStart 컨텍스트 주입 훅 구현 계획

---

## 개요

subagent-context는 코드 작업 서브에이전트 시작 시 프로젝트별 에러 패턴과 AI 분석 규칙을 주입하는 SubagentStart 훅이다. 비코드 에이전트는 건너뛰어 불필요한 컨텍스트 주입을 방지한다.

---

## 기술 결정

### 결정 1: CODE_AGENTS 화이트리스트 방식

**근거:** 코드 에이전트를 명시적으로 나열하여 의도하지 않은 에이전트에 컨텍스트가 주입되는 것을 방지. 새 에이전트 추가 시 목록 업데이트 필요하지만, 안전성이 우선.

**대안 검토:**
- 블랙리스트 방식: 새 에이전트 유형 추가 시 자동 포함되어 불필요한 주입 위험
- agent_type 접두사 기반: 네이밍 규칙에 의존하여 불안정

### 결정 2: includes() 기반 매칭

**근거:** OMC 등에서 `oh-my-claudecode:executor-high` 형태로 접두사가 붙을 수 있으므로, 정확 매치 대신 `includes()` 사용.

### 결정 3: 500자 컨텍스트 제한

**근거:** 서브에이전트의 컨텍스트 윈도우를 과도하게 점유하지 않기 위한 설계 원칙. DESIGN.md 명세 준수.

---

## 구현 단계

### Phase 1: 스캐폴드 및 에이전트 필터링

훅 스크립트 파일 생성, CODE_AGENTS 필터링 로직

**산출물:**
- [ ] `hooks/subagent-context.mjs` 파일 생성 (import, CODE_AGENTS 상수, try-catch)
- [ ] agent_type 필터링 로직 (includes 매칭)
- [ ] 비코드 에이전트 즉시 종료 테스트

### Phase 2: 에러 패턴 + AI 규칙 주입

프로젝트별 에러 조회 및 AI 분석 캐시 조회

**산출물:**
- [ ] 프로젝트별 최근 에러 3건 조회 + searchErrorKB() 해결 이력 병합
- [ ] getCachedAnalysis(48) → claude_md 타입 규칙 최대 3건 추출
- [ ] 전체 컨텍스트 500자 절단 로직
- [ ] 각 기능별 단위 테스트

### Phase 3: 통합 및 출력 테스트

전체 파이프라인 통합 및 stdout JSON 출력 검증

**산출물:**
- [ ] hookSpecificOutput JSON 포맷 검증
- [ ] 에러 패턴 + AI 규칙 동시 존재 시 결합 테스트
- [ ] 컨텍스트 비어있을 때 무출력 테스트
- [ ] exit 0 보장 테스트

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| CODE_AGENTS 목록 갱신 누락 | 🟡 MEDIUM | 상수로 관리하여 변경 포인트 단일화; 문서에 갱신 절차 명시 |
| ai-analyzer 모듈 import 실패 (미구현 상태) | 🟡 MEDIUM | try-catch로 흡수, AI 규칙 없이 에러 패턴만 주입 |
| 500자 절단으로 중요 정보 손실 | 🟢 LOW | 에러 3건 + 규칙 3건이면 통상 500자 이내; 절단 시에도 가장 중요한 정보(에러)가 먼저 배치 |

---

## 테스트 전략

### 단위 테스트

- 에이전트 필터링: 코드 에이전트, 비코드 에이전트, 복합 타입명
- 에러 패턴 주입: 에러 있음/없음, KB 매치 있음/없음
- AI 규칙 주입: 캐시 유효/만료/없음, 프로젝트 필터링
- 500자 절단: 초과/미만 케이스

### 통합 테스트

- error-kb + ai-analyzer + log-writer 3개 모듈 연동 파이프라인
- 훅 등록 후 실제 SubagentStart 이벤트 처리

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] error-kb, ai-analyzer 구현 완료 후 착수 (의존성)
3. [ ] Phase 1부터 순차 구현 시작
