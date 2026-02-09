# 체크리스트: prompt-logger

## 스펙 완성도
- [x] 모든 REQ(101~106)에 GIVEN-WHEN-THEN 시나리오 포함
- [x] RFC 2119 키워드 적절히 사용 (SHALL 중심)
- [x] depends 필드 정확 (log-writer, skill-matcher)

## DESIGN.md 일치
- [x] REQ-DC-101: 프롬프트 수집 (type: 'prompt', text, charCount) — DESIGN.md 8.2절 v6 확장 버전 일치
- [x] REQ-DC-102: 프라이버시 보호 (collectPromptText, stripPrivateTags 순서) — DESIGN.md 4.3절 처리 순서 일치
- [x] REQ-DC-103: 스킬 사용 이벤트 기록 (슬래시 커맨드 감지, loadSkills 검증) — DESIGN.md 8.2절 v7 P5 일치
- [x] REQ-DC-104: Non-blocking 실행 (exit 0) — Constitution 2.1절 일치
- [x] REQ-DC-105: 시스템 활성화 체크 (isEnabled) — DESIGN.md 공통 패턴 일치
- [x] REQ-DC-106: 스킬 자동 감지 (matchSkill, 2초 타임아웃, 벡터/키워드 폴백) — DESIGN.md 8.2절 v9 타임아웃 일치

## 교차 참조
- [x] db.mjs insertEvent로 이벤트 기록
- [x] skill-matcher.mjs loadSkills/matchSkill 사용
- [x] stripPrivateTags가 db.mjs에 구현되어 있음
- [x] session-summary에서 프롬프트 수집 여부 확인

## 테스트 계획
- [x] 일반 프롬프트 기록 (prompt 이벤트)
- [x] collectPromptText=false 시 [REDACTED] 치환
- [x] stripPrivateTags <private> 태그 제거
- [x] 슬래시 커맨드 실제 스킬인 경우 skill_used 이벤트 추가 기록
- [x] 슬래시 접두사이지만 실제 스킬 아닌 경우 skill_used 미기록
- [x] matchSkill 벡터 매칭 성공 시 additionalContext 출력
- [x] matchSkill 키워드 폴백 매칭 성공 시 additionalContext 출력
- [x] matchSkill 2초 타임아웃 (Promise.race)
- [x] 스킬 미존재 시 matchSkill 호출 건너뛰기
- [x] stdin 파싱 실패 시 exit 0
- [x] isEnabled false 시 즉시 exit 0

## 구현 주의사항
- **v6 확장 버전 구현**: DESIGN.md 4.3절(Phase 1)이 아닌 8.2절(v6 확장) 구현 — Phase 1과 v6 병합하지 말 것
- **charCount는 처리 후**: collectPromptText 처리 → stripPrivateTags 적용 → 결과 텍스트의 길이 사용
- **loadSkills()는 skill-matcher에서 import**: 전역 + 프로젝트 스킬 모두 로드
- **matchSkill() 2단계**: (1) 벡터 유사도 distance < 0.76, (2) 키워드 50% 매칭 폴백
- **additionalContext 형식**: hookEventName: 'UserPromptSubmit', 스킬 사용 제안 메시지
- **JSONL 제거**: 저장소는 SQLite events 테이블, insertEvent() 사용
