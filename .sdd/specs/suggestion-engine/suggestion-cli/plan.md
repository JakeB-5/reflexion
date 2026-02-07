---
feature: suggestion-cli
created: 2026-02-07
status: draft
---

# 구현 계획: suggestion-cli

> AI 분석 결과로 생성된 제안을 적용(apply)하거나 거부(dismiss)하는 CLI 도구

---

## 개요

`bin/apply.mjs`와 `bin/dismiss.mjs` 두 CLI 스크립트를 구현한다. apply는 analysis-cache.json에서 제안을 읽어 유형별(skill/claude_md/hook) 적용 로직을 실행하고, dismiss는 거부를 feedback-tracker에 기록한다.

---

## 기술 결정

### 결정 1: 동기 파일 I/O 사용

**근거:** CLI 도구는 단발 실행이므로 비동기 처리의 이점이 없음. `writeFileSync`, `readFileSync` 등 동기 API로 구현 단순화.

**대안 검토:**
- 비동기 I/O (`fs/promises`) — 불필요한 복잡성 추가
- 스트림 기반 — CLI 단발 실행에 과도함

### 결정 2: 인덱스(번호) 기반 제안 선택

**근거:** DESIGN.md 참조 구현과 동일. 사용자가 `analyze.mjs` 출력의 번호로 제안을 선택하는 방식이 직관적.

**대안 검토:**
- ID 기반 선택 — 긴 문자열 입력이 불편
- 대화형 선택 — 순수 Node.js로 구현 시 복잡

---

## 구현 단계

### Phase 1: 스캐폴딩

`bin/apply.mjs`와 `bin/dismiss.mjs`의 기본 구조 생성. CLI 인자 파싱, 캐시 조회, 에러 처리 프레임워크.

**산출물:**
- [ ] `bin/apply.mjs` 기본 구조 (인자 파싱 + 캐시 조회)
- [ ] `bin/dismiss.mjs` 기본 구조 (인자 파싱 + recordFeedback 호출)
- [ ] ai-analyzer의 `getCachedAnalysis()` import 연동

### Phase 2: 핵심 적용 로직

3가지 제안 유형별 적용 함수 구현: `applySkill()`, `applyClaudeMd()`, hook 처리.

**산출물:**
- [ ] `applySkill()` — `.claude/commands/<name>.md` 생성 (전역/프로젝트)
- [ ] `applyClaudeMd()` — CLAUDE.md 규칙 추가 (중복 검사 포함)
- [ ] hook 적용 — `hooks/auto/workflow-<id>.mjs` 생성 + `--apply` 시 settings.json 등록
- [ ] `recordFeedback()` 연동 (적용/거부 모두)

### Phase 3: 테스트 및 검증

각 유형별 적용/거부 시나리오 테스트.

**산출물:**
- [ ] 스킬 생성 테스트 (프로젝트/전역)
- [ ] CLAUDE.md 규칙 추가 테스트 (신규/중복)
- [ ] 훅 생성 테스트 (수동/자동 등록)
- [ ] dismiss 테스트
- [ ] 에러 케이스 테스트 (캐시 없음, 범위 초과, ID 미지정)

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| settings.json 덮어쓰기 시 기존 설정 손실 | HIGH | 기존 파일 읽기 → 병합 → 쓰기 패턴 적용 |
| CLAUDE.md 포맷 깨짐 | MEDIUM | 섹션 존재 여부 확인 후 조건부 추가 |
| analysis-cache.json 스키마 변경 | LOW | ai-analyzer 모듈의 getCachedAnalysis() 사용으로 스키마 의존성 격리 |

---

## 테스트 전략

### 단위 테스트

- `applySkill()`: 파일 생성 확인, 전역/프로젝트 경로 분기
- `applyClaudeMd()`: 규칙 추가, 중복 방지, 섹션 자동 생성
- hook 적용: 스크립트 생성, settings.json 병합
- 커버리지 목표: 80% 이상

### 통합 테스트

- apply.mjs: 전체 실행 흐름 (캐시 조회 → 유형 분기 → 파일 생성 → 피드백 기록)
- dismiss.mjs: 전체 실행 흐름 (인자 파싱 → 피드백 기록)

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] feedback-tracker 모듈 구현 완료 확인 (의존성)
3. [ ] ai-analyzer 모듈의 getCachedAnalysis() 인터페이스 확정
4. [ ] 구현 시작
