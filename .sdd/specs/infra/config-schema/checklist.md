---
feature: config-schema
created: 2026-02-09
---

# 검증 체크리스트: config-schema

## 스펙 완성도

- [ ] config.json 전체 스키마 정의 (최상위 8개 필드 + embedding 중첩 6개 + server 중첩 3개)
- [ ] `loadConfig()` 함수 인터페이스 및 시나리오 정의
- [ ] `isEnabled()` 함수 인터페이스 및 시나리오 정의
- [ ] 기본값 상수 11개 정의
- [ ] 필드별 타입 검증 규칙 테이블
- [ ] install-cli 초기 설정 생성 시나리오

## DESIGN.md 정합성

- [ ] 9.6절 config.json 스키마와 일치
- [ ] 4.2절 `loadConfig()` / `isEnabled()` 구현과 일치
- [ ] install-cli의 config.json 초기화 코드와 일치
- [ ] `pruneOldEvents()`에서의 `retentionDays` 기본값 패턴과 일치

## SDD 규약 준수

- [ ] RFC 2119 키워드 사용 (SHALL, SHOULD)
- [ ] GIVEN-WHEN-THEN 시나리오 형식
- [ ] REQ 번호 체계 (REQ-INF-1xx)
- [ ] 문서 언어: 한글 / 코드: 영어

## 의존성 확인

- [ ] log-writer 스펙(REQ-DC-005)과의 관계 명시
- [ ] install-cli 스펙과의 관계 명시
- [ ] 다른 훅 스펙에서의 `isEnabled()` 사용 패턴 일관성

## 테스트 시나리오 커버리지

- [ ] 정상 로딩 시나리오
- [ ] config.json 미존재 시나리오
- [ ] JSON 파싱 에러 시나리오
- [ ] `isEnabled()` true/false 분기
- [ ] 부분 설정 + 기본값 병합
- [ ] 잘못된 타입 입력 시 기본값 대체
