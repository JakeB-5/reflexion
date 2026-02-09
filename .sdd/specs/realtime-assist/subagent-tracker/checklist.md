# 체크리스트: subagent-tracker

## 스펙 완성도
- [ ] 모든 REQ에 GIVEN-WHEN-THEN 시나리오 포함
- [ ] RFC 2119 키워드 적절히 사용 (SHALL, SHALL NOT, SHOULD, MAY)
- [ ] depends 필드 정확 (`data-collection/log-writer`)

## DESIGN.md 일치
- [ ] REQ-RA-201: SubagentStop 이벤트 기록 필드 (type, ts, sessionId, project, projectPath, agentId, agentType)가 DESIGN.md 9.4.1과 일치
- [ ] REQ-RA-202: SQL 집계 쿼리 전략 (subagent-stats.jsonl 파일 사용 금지)이 DESIGN.md 9.4.2와 일치
- [ ] REQ-RA-203: 비차단 실행 보장 (try-catch, exit 0, 예외 흡수)이 DESIGN.md 6장 훅 원칙과 일치
- [ ] REQ-RA-204: isEnabled() 체크가 DESIGN.md 6장 훅 원칙과 일치
- [ ] v9 변경사항: success 필드 제거 사유 (SubagentStop API 한계)가 DESIGN.md와 일치
- [ ] 훅 등록 형식 (settings.json)이 DESIGN.md 8.7과 일치
- [ ] SubagentStop stdin 필드 (agent_id, agent_type, session_id, cwd, agent_transcript_path)가 DESIGN.md 8.5와 일치

## 교차 참조
- [ ] data-collection/log-writer의 insertEvent() 인터페이스와 호환 (events 테이블 구조)
- [ ] data-collection/log-writer의 readStdin(), isEnabled(), getProjectName(), getProjectPath() 함수 의존성 명시
- [ ] events 테이블 스키마 (type='subagent_stop', data 컬럼 JSON 구조)와 타 스펙 호환
- [ ] ai-analyzer에서 SQL 집계 쿼리로 서브에이전트 통계 조회 시나리오 일치

## 테스트 계획
- [ ] SubagentStop 이벤트 기록 통합 테스트: 정상 이벤트 INSERT, stdin 필드 누락 시 가능한 필드만 기록
- [ ] SQL 집계 쿼리 테스트: agent_type별 사용 횟수 조회, 데이터 없을 때 빈 결과
- [ ] 비차단 실행 테스트: stdin 파싱 실패 시 exit 0, DB 쓰기 실패 시 exit 0
- [ ] isEnabled() 체크 테스트: 시스템 비활성화 시 즉시 종료
- [ ] 성능 테스트: 훅 실행 시간 2초 이내
- [ ] 데이터 무결성 테스트: events 테이블 스키마 준수, data 컬럼 유효한 JSON
- [ ] success 필드 부재 검증: events 테이블에 success 필드 없음 확인 (v9 회귀 방지)
