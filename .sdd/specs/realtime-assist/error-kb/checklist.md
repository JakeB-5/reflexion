# 체크리스트: error-kb

## 스펙 완성도
- [ ] 모든 REQ에 GIVEN-WHEN-THEN 시나리오 포함
- [ ] RFC 2119 키워드 적절히 사용 (SHALL, SHOULD, MAY)
- [ ] depends 필드 정확 (`data-collection/log-writer`)

## DESIGN.md 일치
- [ ] REQ-RA-001: normalizeError() 정규화 규칙 (경로, 숫자, 문자열, 200자 절단)이 DESIGN.md 9.2.1과 일치
- [ ] REQ-RA-002: Strong-signal shortcut 패턴 (정확 매치 → 접두사 매치 → 벡터 검색) 3단계 전략이 DESIGN.md 9.2.2와 일치
- [ ] REQ-RA-003: UPSERT 패턴 (ON CONFLICT DO UPDATE)이 DESIGN.md 9.2.3과 일치
- [ ] REQ-RA-004: error_kb 테이블 스키마 (ts, error_normalized UNIQUE, error_raw, resolution, resolved_by, tool_sequence, use_count, last_used)가 DESIGN.md 9.1과 일치
- [ ] REQ-RA-005: 배치 임베딩 생성 시점 (SessionEnd)과 최대 50건 처리가 DESIGN.md 9.2.4와 일치
- [ ] 벡터 임베딩 차원 (384) 및 cosine distance 임계값 (0.76)이 DESIGN.md와 일치
- [ ] sqlite-vec 확장 사용 및 vec_error_kb 가상 테이블 구조가 DESIGN.md 9.1과 일치

## 교차 참조
- [ ] data-collection/log-writer의 insertEvent() 인터페이스와 호환 (events 테이블 구조)
- [ ] data-collection/log-writer의 generateEmbeddings() 함수 의존성 명시
- [ ] data-collection/log-writer의 vectorSearch() 함수 의존성 명시
- [ ] 타 스펙에서 normalizeError() import 시 이 모듈이 Single Owner임을 명시
- [ ] error-logger, tool-logger 훅에서 recordResolution() 호출 시나리오 일치
- [ ] session-summary 훅에서 generateErrorEmbeddings() 호출 시나리오 일치

## 테스트 계획
- [ ] normalizeError() 단위 테스트: 경로 치환, 숫자 치환, 따옴표 문자열 치환, 200자 절단
- [ ] searchErrorKB() 통합 테스트: 정확 텍스트 매치 (~1ms), 접두사 매치 (70% 길이 비율), 벡터 검색 fallback (distance < 0.76)
- [ ] recordResolution() 통합 테스트: 새 엔트리 INSERT, 기존 엔트리 UPSERT (use_count 증가)
- [ ] generateErrorEmbeddings() 통합 테스트: 임베딩 없는 엔트리 배치 생성, 부분 실패 시 나머지 계속 처리
- [ ] error_kb 테이블 스키마 검증: UNIQUE 제약 조건, NOT NULL 제약 조건
- [ ] 성능 테스트: searchErrorKB() 50ms 이내, 10,000건 이상에서 성능 저하 없음
- [ ] 안정성 테스트: DB 부재 시 null 반환, 예외 발생 시 프로세스 중단 없음
