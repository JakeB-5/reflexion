# session-start-hook 검증 체크리스트

## REQ-SSH-001: 시스템 활성화 확인

- [ ] `isEnabled()` 함수로 활성화 여부 확인
- [ ] 비활성 시 무출력으로 exit code 0
- [ ] 활성 시 캐시 주입 로직 계속 실행

## REQ-SSH-002: 캐시된 분석 결과 주입

- [ ] `getCachedAnalysis(24, project)` 호출로 24시간 이내 캐시 조회
- [ ] stdin의 `cwd`에서 `getProjectPath()` → `getProjectName()` 추출
- [ ] 제안이 존재하면 최대 3개 `additionalContext` 주입
- [ ] 캐시 없음 시 제안 미주입
- [ ] 캐시 만료(30시간 전) 시 제안 미주입

## REQ-SSH-003: 제안 메시지 포맷팅

- [ ] 포맷: `- [type] summary [id: suggest-N]`
- [ ] 헤더: `[Self-Generation] AI 패턴 분석 결과:`
- [ ] 적용 안내: `node ~/.self-generation/bin/apply.mjs <번호>`
- [ ] 거부 안내: `node ~/.self-generation/bin/dismiss.mjs <id>`

## REQ-SSH-004: 이전 세션 컨텍스트 주입

- [ ] `queryEvents({ type: 'session_summary', projectPath, limit: 1 })` 조회
- [ ] 프롬프트 수, 도구 사용 횟수 포함
- [ ] 마지막 프롬프트 포함
- [ ] 편집 중이던 파일 포함
- [ ] 미해결 에러 포함
- [ ] 주요 도구 상위 3개 포함
- [ ] 이전 세션 없음 시 미주입

## REQ-SSH-005: Non-blocking 실행 보장

- [ ] 모든 오류 상황에서 exit code 0
