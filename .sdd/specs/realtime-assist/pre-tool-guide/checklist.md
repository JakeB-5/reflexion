# 체크리스트: pre-tool-guide

## 스펙 완성도
- [ ] 모든 REQ에 GIVEN-WHEN-THEN 시나리오 포함
- [ ] RFC 2119 키워드 적절히 사용 (SHALL, SHALL NOT, SHOULD, MAY)
- [ ] depends 필드 정확 (`data-collection/log-writer, realtime-assist/error-kb`)

## DESIGN.md 일치
- [ ] REQ-RA-301: Edit/Write 도구 파일 관련 에러 이력 주입 (error_kb LIKE 쿼리, 최근 2건)이 DESIGN.md 9.6.1과 일치
- [ ] REQ-RA-302: Bash 도구 세션 내 에러 이력 주입 (events 테이블 + error_kb 정확 매치)이 DESIGN.md 9.6.2와 일치
- [ ] REQ-RA-303: Task 도구 서브에이전트 실패율 경고 비활성화 (v9)가 DESIGN.md와 일치
- [ ] REQ-RA-304: 출력 형식 (hookEventName="PreToolUse", additionalContext)이 DESIGN.md 8.5와 일치
- [ ] REQ-RA-305: 비차단 실행 보장 (try-catch, exit 0)이 DESIGN.md 6장 훅 원칙과 일치
- [ ] 훅 등록 형식 (matcher="Edit|Write|Bash|Task")이 DESIGN.md 8.7과 일치
- [ ] 설계 원칙: 벡터 검색 회피, 텍스트 매칭만 사용 (동기 블로킹 훅)이 DESIGN.md 9.6과 일치

## 교차 참조
- [ ] data-collection/log-writer의 queryEvents(), getDb(), readStdin(), isEnabled() 함수 의존성 명시
- [ ] realtime-assist/error-kb의 error_kb 테이블 스키마 (error_normalized, resolution) 일치
- [ ] error-kb의 searchErrorKB() 사용 금지 명시 (벡터 검색 회피)
- [ ] tool-logger 훅에서 기록한 tool_error 이벤트 구조와 호환
- [ ] subagent-tracker의 success 필드 제거와 동기화 (REQ-RA-303 비활성화 배경)

## 테스트 계획
- [ ] Edit/Write 도구 테스트: 파일 관련 에러 이력 조회 (error_kb LIKE 쿼리), 이력 없을 때 가이드 생략, resolution JSON 파싱
- [ ] Bash 도구 테스트: 세션 내 Bash 에러 조회 (events 테이블), error_kb 정확 매치, 에러 없거나 KB 매치 없을 때 가이드 생략
- [ ] Task 도구 테스트: 서브에이전트 실패율 경고 비활성화 확인 (v9)
- [ ] 출력 형식 테스트: hookEventName="PreToolUse" 검증, additionalContext 개행 구분, 가이드 없을 때 stdout 없이 종료
- [ ] 비차단 실행 테스트: DB 접근 실패 시 exit 0
- [ ] 성능 테스트: 훅 실행 시간 2초 이내, 벡터 검색 회피로 지연 최소화
- [ ] matcher 필터링 테스트: Edit/Write/Bash/Task만 훅 실행, 다른 도구(Read, Grep) 필터링
