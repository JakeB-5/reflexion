# 체크리스트: feedback-tracker

## 스펙 완성도
- [ ] 모든 REQ에 GIVEN-WHEN-THEN 시나리오 포함
- [ ] RFC 2119 키워드 적절히 사용 (SHALL/SHOULD)
- [ ] depends 필드 정확: data-collection/log-writer, realtime-assist/skill-matcher

## DESIGN.md 일치

### REQ-SE-101: 피드백 기록
- [ ] `recordFeedback(suggestionId, action, details)` 함수 export
- [ ] `action`은 'accepted', 'rejected', 'dismissed' 중 하나
- [ ] INSERT SQL: `INSERT INTO feedback (v, ts, suggestion_id, action, suggestion_type, summary) VALUES (1, ?, ?, ?, ?, ?)`
- [ ] `details.suggestionType` → `suggestion_type` 매핑
- [ ] `details.summary` → `summary` 매핑
- [ ] details 미지정 시 suggestion_type과 summary를 null로 기록

### REQ-SE-102: 피드백 요약 생성
- [ ] `getFeedbackSummary()` 함수 export
- [ ] JavaScript 집계 방식 사용 (SQL 집계 아님): `SELECT * FROM feedback ORDER BY ts ASC` 후 JavaScript 필터링
- [ ] 반환 필드: total, acceptedCount, rejectedCount, rate, recentRejections, recentAcceptances, skillUsageRate, ruleEffectiveness, staleSkills
- [ ] rate 계산: `acceptedCount / total` (0~1)
- [ ] recentRejections/recentAcceptances: ASC 정렬 후 `.slice(-10)` 사용, `e.summary || e.suggestion_id`로 매핑
- [ ] feedback 테이블 비어있으면 `null` 반환
- [ ] DB 접근 실패 시 예외 throw하지 않고 `null` 반환

### REQ-SE-103: 스킬 사용률 계산 (내부 함수)
- [ ] `calcSkillUsageRate()` 내부 함수 (export 안 함)
- [ ] SQL COUNT로 `skill_used` 이벤트 수 / `skill_created` 이벤트 수 계산
- [ ] `skill_created`가 0이면 `null` 반환
- [ ] DB 접근 실패 시 `null` 반환

### REQ-SE-104: 규칙 효과성 측정 (내부 함수)
- [ ] `calcRuleEffectiveness()` 내부 함수 (export 안 함)
- [ ] `tool_error` 이벤트 전체 수와 최근 7일 내 발생 수 반환
- [ ] 최근 7일 기준: `new Date(Date.now() - 7 * 86400000).toISOString()` 계산 후 SQL 파라미터로 전달
- [ ] 반환: `{ totalErrors, recentErrors }`
- [ ] DB 접근 실패 시 `null` 반환

### REQ-SE-105: 미사용 스킬 탐지 (내부 함수)
- [ ] `findStaleSkills(days, projectPath = null)` 내부 함수 (export 안 함)
- [ ] 기본 30일 동안 사용되지 않은 스킬 탐지
- [ ] `loadSkills(projectPath)`로 등록된 스킬 목록 가져오기
- [ ] 각 스킬에 대해 `events` 테이블에서 최근 사용 이력 조회: `json_extract(data, '$.skillName') = ?`
- [ ] 기간 기준: `new Date(Date.now() - days * 86400000).toISOString()` 계산
- [ ] 마지막 사용 시점이 없거나 threshold보다 이전이면 미사용으로 판정
- [ ] DB 접근 실패 시 빈 배열 `[]` 반환

### REQ-SE-106: 피드백 → AI 프롬프트 주입
- [ ] `getFeedbackSummary()` 반환값이 AI 분석 프롬프트(`analyze.md`)에 주입되는 형식 정의
- [ ] 프롬프트 주입은 ai-analyzer 모듈이 담당 (feedback-tracker는 데이터 제공만)

## 교차 참조

### data-collection/log-writer 인터페이스
- [ ] `getDb()` 함수로 DB 접근
- [ ] `queryEvents()` 함수 사용 가능

### realtime-assist/skill-matcher 인터페이스
- [ ] `loadSkills(projectPath)` 함수 시그니처 일치
- [ ] 반환 스킬 객체에 `name` 필드 포함

### ai-analysis/ai-analyzer 연동
- [ ] `getFeedbackSummary()` 반환 객체가 analyze.md 템플릿에 주입되는 형식 일치

## 테스트 계획

### 피드백 기록 테스트
- [ ] 채택 피드백 기록 (`action: 'accepted'`) 검증
- [ ] 거부 피드백 기록 (`action: 'rejected'`) 검증
- [ ] details 미지정 시 null 필드로 기록 검증
- [ ] feedback 테이블 INSERT-only 정책 검증

### 피드백 요약 생성 테스트
- [ ] 정상 요약 생성 (total, acceptedCount, rejectedCount, rate 계산) 검증
- [ ] feedback 테이블 비어있을 때 `null` 반환 검증
- [ ] 최근 이력 제한 (recentRejections/recentAcceptances 최대 10건) 검증
- [ ] DB 접근 실패 시 `null` 반환 검증

### 스킬 사용률 계산 테스트
- [ ] skill_used 이벤트 10건, skill_created 이벤트 4건일 때 2.5 반환 검증
- [ ] skill_created 이벤트 없을 때 `null` 반환 검증
- [ ] DB 접근 실패 시 `null` 반환 검증

### 규칙 효과성 측정 테스트
- [ ] 전체 tool_error 20건, 최근 7일 내 3건일 때 `{ totalErrors: 20, recentErrors: 3 }` 반환 검증
- [ ] DB 접근 실패 시 `null` 반환 검증

### 미사용 스킬 탐지 테스트
- [ ] 스킬 3개 중 2개가 30일 내 미사용일 때 2개 스킬명 배열 반환 검증
- [ ] projectPath 지정 시 프로젝트별 스킬만 조회 검증
- [ ] 모든 스킬이 활발히 사용 중일 때 빈 배열 반환 검증
- [ ] DB 접근 실패 시 빈 배열 반환 검증

### AI 프롬프트 주입 테스트
- [ ] getFeedbackSummary() 반환값이 ai-analyzer의 analyze.md 템플릿에 정확히 주입되는지 검증
- [ ] 피드백 없을 때 프롬프트 섹션 생략 검증

## 성능 요구사항
- [ ] `recordFeedback()`: 10ms 이내 (SHOULD) — 단일 INSERT 쿼리
- [ ] `getFeedbackSummary()`: 50ms 이내 (SHOULD) — SELECT + JavaScript 집계
- [ ] `calcSkillUsageRate()`, `calcRuleEffectiveness()`: 100ms 이내 (SHOULD) — SQL COUNT 쿼리

## 데이터 무결성 요구사항
- [ ] feedback 테이블 INSERT-only (UPDATE/DELETE 금지)
- [ ] 모든 레코드에 `v` 컬럼으로 스키마 버전 1 포함
- [ ] SQLite 트랜잭션으로 데이터 무결성 보장

## 에러 처리 요구사항
- [ ] recordFeedback: 예외 발생 시 조용히 실패 (try-catch)
- [ ] getFeedbackSummary: 예외 발생 시 `null` 반환
- [ ] calcSkillUsageRate: 예외 발생 시 `null` 반환
- [ ] calcRuleEffectiveness: 예외 발생 시 `null` 반환
- [ ] findStaleSkills: 예외 발생 시 빈 배열 `[]` 반환

## 제약사항 검증
- [ ] `better-sqlite3` 외 추가 npm 패키지 사용 금지 (db.mjs 통해 간접 사용)
- [ ] ES Modules 형식 (`.mjs` 확장자)
- [ ] events 테이블 읽기는 `lib/db.mjs`의 `getDb()` 통해 직접 SQL 실행
- [ ] feedback 테이블 쓰기/읽기는 `lib/db.mjs`의 `getDb()` 통해 직접 SQL 실행
- [ ] `loadSkills()`는 `lib/skill-matcher.mjs`에서 import
- [ ] 날짜 비교는 JavaScript `Date.now()` 기반 계산 후 ISO 8601 문자열 변환 사용
