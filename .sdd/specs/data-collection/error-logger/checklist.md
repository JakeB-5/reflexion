---
feature: error-logger
created: 2026-02-09
---

# 검증 체크리스트: error-logger

## 스펙 완전성

- [ ] REQ-DC-301: 에러 수집 — events 테이블에 tool_error 타입 기록
- [ ] REQ-DC-302: 에러 정규화 (normalizeError) — error-kb.mjs에서 import
- [ ] REQ-DC-303: 에러 KB 실시간 검색 및 컨텍스트 주입 — searchErrorKB, additionalContext
- [ ] REQ-DC-304: Non-blocking 실행 보장 — KB 검색 실패 시에도 exit 0
- [ ] REQ-DC-305: 시스템 활성화 체크 — isEnabled() 확인

## Constitution 준수

- [ ] 비차단 훅: 모든 예외 포착, exit 0 보장
- [ ] 프라이버시: 에러 메시지 정규화 (경로/숫자/문자열 치환)
- [ ] 전역 우선: events 테이블에 project_path 기반 저장
- [ ] 스키마 버전: v:1 포함
- [ ] 명세 우선: 본 스펙 문서 작성 완료
- [ ] RFC 2119: SHALL/SHOULD 키워드 사용

## DESIGN.md 일관성

- [ ] 8.1절 v6 확장 버전 구현 (Phase 1 4.5절이 아님)
- [ ] normalizeError() 단일 소유자 원칙 (error-kb.mjs)
- [ ] searchErrorKB() 사용 (error-kb.mjs)
- [ ] PostToolUseFailure 이벤트 매핑 (tool_name, error, session_id, cwd)
- [ ] additionalContext 출력 형식 일치
- [ ] 2초 타임아웃 적용 (콜드 스타트 방지)
- [ ] errorRaw 500자 제한

## 의존성 검증

- [ ] data-collection/log-writer: insertEvent(), isEnabled(), readStdin()
- [ ] realtime-assist/error-kb: normalizeError(), searchErrorKB()

## GIVEN-WHEN-THEN 시나리오

- [ ] REQ-DC-301: 2개 시나리오 (도구 실패 기록, stdin 매핑)
- [ ] REQ-DC-302: 2개 시나리오 (정규화 함수 사용, 정규화 규칙)
- [ ] REQ-DC-303: 5개 시나리오 (과거 이력 발견, resolution JSON 파싱, 2초 타임아웃, 미발견, 스킵 조건)
- [ ] REQ-DC-304: 2개 시나리오 (KB 검색 실패, 에러 기록 정상)
- [ ] REQ-DC-305: 1개 시나리오 (시스템 비활성화)

## 테스트 시나리오 커버리지

- [ ] 에러 수집: stdin 파싱, 필드 매핑, insertEvent 호출
- [ ] 정규화: normalizeError 호출, 경로/숫자/문자열 치환, 200자 절단
- [ ] KB 검색: searchErrorKB 호출, 벡터 유사도 우선, 텍스트 폴백
- [ ] additionalContext: JSON 출력 형식, resolution 파싱, 표시 형식
- [ ] 타임아웃: Promise.race 2초, 콜드 스타트 시나리오
- [ ] 에러 처리: try-catch, exit 0 보장
- [ ] isEnabled: config.enabled=false 시 즉시 종료

## 비고

- v6 확장 버전이 최종 구현 (Phase 1 + v6 병합 금지)
- normalizeError는 error-kb.mjs에서 단일 관리
- 실시간 에러 안내로 문제 해결 속도 향상
