# 체크리스트: suggestion-cli

## 스펙 완성도
- [ ] 모든 REQ에 GIVEN-WHEN-THEN 시나리오 포함
- [ ] RFC 2119 키워드 적절히 사용 (SHALL/SHOULD/MAY)
- [ ] depends 필드 정확: ai-analysis/ai-analyzer, suggestion-engine/feedback-tracker, data-collection/log-writer

## DESIGN.md 일치

### REQ-SE-001: CLI 인자 파싱 및 제안 조회
- [ ] `<suggestion-number>`, `--global`, `--project`, `--apply` 플래그 파싱 구현
- [ ] `getCachedAnalysis(168, project)` 호출로 7일 TTL 캐시 조회
- [ ] 캐시 없음/제안 비어있음 시 "분석 결과가 없습니다. 먼저 node ~/.self-generation/bin/analyze.mjs 를 실행하세요." 오류 출력 및 exit 1
- [ ] 제안 번호 범위 초과 시 "유효한 범위: 1-N" 오류 출력 및 exit 1
- [ ] 번호 미지정/비정수 시 "사용법: node ~/.self-generation/bin/apply.mjs <번호> [--global]" 출력 및 exit 1

### REQ-SE-002: 스킬 제안 적용
- [ ] `type: 'skill'` 제안 시 `.claude/commands/<skillName>.md` 파일 생성
- [ ] `skillName` 없으면 `'auto-skill'` 기본값 사용
- [ ] `--global` 플래그 지정 시 `~/.claude/commands/` 경로 사용
- [ ] 디렉터리 없으면 재귀적으로 생성 (`fs.mkdirSync(dir, { recursive: true })`)
- [ ] 스킬 파일 구조: `# /<skillName>` 제목, `## 감지된 패턴` 섹션 (suggestion.evidence), `## 실행 지침` 섹션 (suggestion.action)

### REQ-SE-003: CLAUDE.md 규칙 제안 적용
- [ ] `type: 'claude_md'` 제안 시 대상 CLAUDE.md에 규칙 추가
- [ ] `suggestion.rule` 또는 `suggestion.summary`에서 규칙 텍스트 가져오기
- [ ] 추가 전 동일 규칙 중복 검사 (이미 존재하면 "이미 동일한 규칙이 존재합니다." 출력하고 파일 수정 안 함)
- [ ] `## 자동 감지된 규칙` 섹션 없으면 생성
- [ ] `--global` 시 `~/.claude/CLAUDE.md`, 미지정 시 `<cwd>/.claude/CLAUDE.md`
- [ ] 대상 디렉터리 없으면 재귀 생성

### REQ-SE-004: 훅 워크플로우 제안 적용
- [ ] `type: 'hook'` 제안 시 `~/.self-generation/hooks/auto/workflow-<id>.mjs` 파일 생성
- [ ] 디렉터리 없으면 재귀 생성
- [ ] `suggestion.hookCode` 존재하면 파일로 저장
- [ ] `--apply` 플래그 지정 시 `~/.claude/settings.json`에 훅 자동 등록 (기존 settings.json 구조 보존)
- [ ] `--apply` 없으면 등록 안내와 자동 등록 명령어 출력
- [ ] `hookEvent` 미지정 시 `'PostToolUse'` 기본값 사용
- [ ] `hookCode` 없으면 "훅 코드 미생성 — 프롬프트 템플릿에 hookCode 필드를 요청하세요" 경고 출력

### REQ-SE-005: 제안 거부 기록
- [ ] `bin/dismiss.mjs`가 `recordFeedback(id, 'rejected', { suggestionType: 'unknown' })` 호출
- [ ] 거부 확인 메시지 출력: "제안 거부 기록됨: <id>"
- [ ] 향후 AI 분석 제외 안내 출력: "이 패턴은 향후 AI 분석 시 제외 컨텍스트로 전달됩니다."
- [ ] ID 미지정 시 "사용법: node ~/.self-generation/bin/dismiss.mjs <suggestion-id>" 출력 및 exit 1

### REQ-SE-006: 피드백 기록 연동
- [ ] 모든 적용(apply) 성공 시 `recordFeedback(id, 'accepted', { suggestionType, summary })` 호출
- [ ] 피드백 기록은 제안 유형별 적용 로직(switch 분기) 이후 공통 실행

### REQ-SE-007: 이벤트 추적
- [ ] 제안 유형이 `'skill'`이면 `type: 'skill_created'` 이벤트 기록
- [ ] 그 외 제안 유형이면 `type: 'suggestion_applied'` 이벤트 기록
- [ ] 이벤트 데이터: `{ v: 1, type, ts, project, data: { suggestionId, suggestionType, scope } }`
- [ ] `project`는 `getProjectName(process.cwd())`로 결정
- [ ] `--global` 플래그 사용 시 `data.scope: 'global'`, 미사용 시 `data.scope: 'project'`

## 교차 참조

### ai-analysis/ai-analyzer 인터페이스
- [ ] `getCachedAnalysis(maxAgeHours, project)` 함수 시그니처 일치
- [ ] 반환 객체의 `suggestions` 배열 구조: `{ id, type, skillName?, rule?, hookCode?, hookEvent?, summary?, evidence?, action? }`

### suggestion-engine/feedback-tracker 인터페이스
- [ ] `recordFeedback(suggestionId, action, details)` 함수 시그니처 일치
- [ ] `action` 값: 'accepted', 'rejected', 'dismissed' 중 하나
- [ ] `details` 객체: `{ suggestionType, summary }` 필드 포함

### data-collection/log-writer 인터페이스
- [ ] `insertEvent(eventData)` 함수 시그니처 일치
- [ ] `getProjectName(projectPath)` 함수 사용

## 테스트 계획

### CLI 인자 파싱 테스트
- [ ] 정상 제안 번호 입력 시 해당 제안 선택 검증
- [ ] `--project` 플래그로 프로젝트 필터링 동작 검증
- [ ] 캐시 없음 시 오류 메시지 출력 검증
- [ ] 범위 초과 번호 입력 시 오류 메시지 출력 검증
- [ ] 번호 미지정/비정수 입력 시 사용법 출력 검증

### 스킬 적용 테스트
- [ ] 프로젝트 스킬 생성 (`<cwd>/.claude/commands/<skillName>.md`) 검증
- [ ] 전역 스킬 생성 (`~/.claude/commands/<skillName>.md`) with `--global` 검증
- [ ] skillName 미지정 시 `auto-skill.md` 파일명 사용 검증
- [ ] 스킬 파일 내용 구조 (제목, 감지된 패턴, 실행 지침) 검증

### CLAUDE.md 규칙 적용 테스트
- [ ] 새 규칙 추가 시 `## 자동 감지된 규칙` 섹션에 추가 검증
- [ ] 중복 규칙 방지 (이미 존재하면 추가 안 함) 검증
- [ ] CLAUDE.md 미존재 시 새 파일 생성 검증
- [ ] `--global` vs 프로젝트 범위 파일 위치 검증

### 훅 워크플로우 적용 테스트
- [ ] 훅 스크립트 파일 생성 (`~/.self-generation/hooks/auto/workflow-<id>.mjs`) 검증
- [ ] `--apply` 없이 실행 시 수동 등록 안내 출력 검증
- [ ] `--apply`로 `settings.json` 자동 등록 (기존 구조 보존) 검증
- [ ] hookCode 미포함 시 경고 메시지 출력 검증

### 거부 기록 테스트
- [ ] 정상 거부 시 `feedback` 테이블에 INSERT 검증
- [ ] 거부 확인 메시지 출력 검증
- [ ] ID 미지정 시 사용법 출력 검증

### 피드백 및 이벤트 추적 테스트
- [ ] 스킬 적용 시 `skill_created` 이벤트 + 피드백 기록 검증
- [ ] CLAUDE.md 적용 시 `suggestion_applied` 이벤트 + 피드백 기록 검증
- [ ] 전역 적용 시 `data.scope: 'global'` 기록 검증
- [ ] 프로젝트 적용 시 `data.scope: 'project'` 기록 검증

## 성능 요구사항
- [ ] CLI 실행 시간 500ms 이내 (SHOULD)
- [ ] 동기 파일 I/O 사용 (Node.js built-in `fs`, `path` 모듈)

## 보안 요구사항
- [ ] 훅 자동 등록(`--apply`) 시 기존 `settings.json` 구조 보존 검증
- [ ] 사용자 승인 없이 자동 적용 발생하지 않음 검증

## 제약사항 검증
- [ ] `better-sqlite3` 외 추가 npm 패키지 사용 금지 (db.mjs 통해 간접 사용)
- [ ] ES Modules 형식 (`.mjs` 확장자) 사용
- [ ] 캐시 조회는 `getCachedAnalysis()` 함수로만 수행 (직접 SQL 접근 금지)
- [ ] `recordFeedback()`는 `feedback-tracker` 모듈에서 import
- [ ] `insertEvent()`, `getProjectName()`은 `db` 모듈에서 import
