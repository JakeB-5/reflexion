# ai-analyzer 검증 체크리스트

## REQ-AA-001: Content-Addressable 입력 해시

- [ ] `computeInputHash(events)` 함수로 SHA-256 해시 계산
- [ ] 해시 입력: `type:ts:session_id:data` 조합
- [ ] 동일 이벤트 배열 → 동일 해시
- [ ] 이벤트 변경 → 해시 변경

## REQ-AA-002: 동기 AI 분석 실행

- [ ] `runAnalysis(options = {})` 동기 실행
- [ ] 프롬프트 5개 미만 시 분석 생략
- [ ] Content-Addressable 캐시 히트 확인
- [ ] `claude --print --model sonnet` 실행
- [ ] clusters/workflows/errorPatterns/suggestions/skill_descriptions 포함
- [ ] `analysis_cache` 테이블에 `input_hash` 함께 저장
- [ ] UPSERT로 기존 캐시 갱신 (행 ID 보존)
- [ ] `claude --print` 실패 시 예외 전파 안함

## REQ-AA-003: 비동기 AI 분석 실행

- [ ] `runAnalysisAsync(options)` detached 프로세스로 실행
- [ ] 부모 프로세스 블로킹 방지

## REQ-AA-004: 프롬프트 구성

- [ ] `prompts/analyze.md` 템플릿 로드
- [ ] 실제 로그 데이터 주입
- [ ] 피드백 이력 주입 (feedback-tracker)
- [ ] 기존 글로벌 스킬 주입 (skill-matcher)
- [ ] 결과 메트릭 주입

## REQ-AA-005: 응답 파싱

- [ ] 코드 펜스 (```json) 응답 처리
- [ ] Raw JSON 응답 처리
- [ ] `JSON.parse()` 실패 시 처리

## REQ-AA-006: 캐시 조회 및 TTL

- [ ] TTL 기반 캐시 만료 확인 (기본 24시간)
- [ ] 프로젝트 필터링으로 크로스 프로젝트 오염 방지
- [ ] 실패 시 null 반환

## REQ-AA-007: 로그 요약

- [ ] 최근 100개 프롬프트로 제한
- [ ] 도구 시퀀스 집계 (예: Grep→Read→Edit)
- [ ] 에러 압축
- [ ] 성능: 10K 레코드 500ms 이내
