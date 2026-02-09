# 체크리스트: subagent-context

## 스펙 완성도
- [ ] 모든 REQ에 GIVEN-WHEN-THEN 시나리오 포함
- [ ] RFC 2119 키워드 적절히 사용 (SHALL, SHOULD, MAY)
- [ ] depends 필드 정확 (`data-collection/log-writer, realtime-assist/error-kb, ai-analysis/ai-analyzer`)

## DESIGN.md 일치
- [ ] REQ-RA-401: CODE_AGENTS 목록 (executor, architect, designer, build-fixer 계열)이 DESIGN.md 9.5.1과 일치
- [ ] REQ-RA-402: 에러 패턴 주입 전략 (벡터 검색 회피, error_kb 직접 텍스트 쿼리)이 DESIGN.md 9.5.2와 일치
- [ ] REQ-RA-403: AI 분석 규칙 주입 시 프로젝트 필터링 (getCachedAnalysis 호출 시 project 파라미터)이 DESIGN.md 9.5.3과 일치
- [ ] REQ-RA-404: 컨텍스트 크기 제한 (500자)이 DESIGN.md 9.5.4와 일치
- [ ] REQ-RA-405: 출력 형식 (hookEventName="SubagentStart", additionalContext)이 DESIGN.md 8.5와 일치
- [ ] REQ-RA-406: 비차단 실행 보장 (try-catch, exit 0)이 DESIGN.md 6장 훅 원칙과 일치
- [ ] REQ-RA-407: isEnabled() 체크가 DESIGN.md 6장 훅 원칙과 일치
- [ ] 훅 등록 형식 (settings.json)이 DESIGN.md 8.7과 일치

## 교차 참조
- [ ] data-collection/log-writer의 queryEvents() 인터페이스와 호환
- [ ] realtime-assist/error-kb의 error_kb 테이블 스키마 (error_normalized, resolution) 일치
- [ ] ai-analysis/ai-analyzer의 getCachedAnalysis(hours, project) 함수 시그니처 일치 (project 파라미터 필수)
- [ ] data-collection/log-writer의 getDb(), getProjectName(), getProjectPath(), readStdin(), isEnabled() 함수 의존성 명시
- [ ] prompt-logger 훅에서 코드 에이전트 필터링 로직과 중복 없음 확인

## 테스트 계획
- [ ] 코드 에이전트 필터링 테스트: CODE_AGENTS 포함 시 컨텍스트 주입, 비코드 에이전트 시 즉시 종료, 복합 타입명 (oh-my-claudecode:executor-high) 매칭
- [ ] 에러 패턴 주입 테스트: 프로젝트 에러 3건 조회, error_kb 정확 텍스트 매치, 에러 없을 때 섹션 생략
- [ ] AI 분석 규칙 주입 테스트: 프로젝트 필터 적용, 전역+프로젝트 규칙 혼합, 캐시 만료 시 섹션 생략
- [ ] 컨텍스트 크기 제한 테스트: 500자 초과 시 절단, 주입할 컨텍스트 없을 때 stdout 없이 종료
- [ ] 출력 형식 테스트: hookEventName="SubagentStart" 검증, additionalContext JSON 구조
- [ ] 비차단 실행 테스트: 의존 모듈 로드 실패 시 exit 0
- [ ] isEnabled() 체크 테스트: 시스템 비활성화 시 즉시 종료
- [ ] 성능 테스트: 훅 실행 시간 2초 이내, 벡터 검색 회피로 지연 최소화
