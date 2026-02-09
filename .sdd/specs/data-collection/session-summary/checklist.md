---
feature: session-summary
created: 2026-02-09
---

# 검증 체크리스트: session-summary

## 스펙 완전성

- [ ] REQ-DC-401: 세션 요약 집계 — promptCount/toolCounts/toolSequence/errorCount 등
- [ ] REQ-DC-402: 세션 이벤트 SQL 집계 — queryEvents 사용
- [ ] REQ-DC-403: AI 분석 트리거 조건 — 프롬프트 3개 이상, reason !== 'clear'
- [ ] REQ-DC-404: Non-blocking 실행 보장 — 모든 실패에 exit 0
- [ ] REQ-DC-405: 배치 임베딩 생성 트리거 — detached spawn, unref()
- [ ] REQ-DC-406: 확률적 DB 정리 — 10% 확률 pruneOldEvents
- [ ] REQ-DC-407: 시스템 활성화 체크 — isEnabled() 확인

## Constitution 준수

- [ ] 비차단 훅: 모든 예외 포착, exit 0 보장
- [ ] 전역 우선: events 테이블 SQL 집계
- [ ] 스키마 버전: v:1 포함
- [ ] 명세 우선: 본 스펙 문서 작성 완료
- [ ] RFC 2119: SHALL/SHOULD 키워드 사용

## DESIGN.md 일관성

- [ ] 5.4절 SessionEnd 훅 확장 버전 구현
- [ ] promptCount, toolCounts, toolSequence, errorCount, uniqueErrors 집계
- [ ] lastPrompts 마지막 3개, 100자 제한
- [ ] lastEditedFiles Edit/Write 도구, 중복 제거, 최대 5개
- [ ] reason 필드 기록
- [ ] runAnalysisAsync() 호출 (ai-analyzer.mjs)
- [ ] batch-embeddings.mjs detached spawn

## 의존성 검증

- [ ] data-collection/log-writer: insertEvent(), queryEvents(), pruneOldEvents(), isEnabled(), readStdin()
- [ ] ai-analysis/ai-analyzer: runAnalysisAsync()

## GIVEN-WHEN-THEN 시나리오

- [ ] REQ-DC-401: 7개 시나리오 (정상 요약, toolCounts, toolSequence, uniqueErrors, lastPrompts, lastEditedFiles, reason)
- [ ] REQ-DC-402: 2개 시나리오 (세션 조회, 빈 세션)
- [ ] REQ-DC-403: 4개 시나리오 (분석 트리거, 스킵 조건 2개, 프롬프트 부족)
- [ ] REQ-DC-404: 2개 시나리오 (DB 실패, 분석 실패)
- [ ] REQ-DC-405: 2개 시나리오 (detached spawn, unref 분리)
- [ ] REQ-DC-406: 2개 시나리오 (10% 확률, 실패 무시)
- [ ] REQ-DC-407: 1개 시나리오 (시스템 비활성화)

## 테스트 시나리오 커버리지

- [ ] 세션 집계: queryEvents, type별 필터링, 카운트 계산
- [ ] toolCounts: 도구별 횟수 집계
- [ ] toolSequence: 시간 순서 배열
- [ ] uniqueErrors: 중복 제거 (Set 사용)
- [ ] lastPrompts: slice(-3), 100자 제한
- [ ] lastEditedFiles: Edit/Write 필터, 중복 제거, 최대 5개
- [ ] AI 분석: 조건 분기, runAnalysisAsync 호출
- [ ] 배치 임베딩: spawn detached, unref, stdio ignore
- [ ] DB 정리: Math.random() < 0.1, pruneOldEvents
- [ ] 에러 처리: try-catch, exit 0 보장
- [ ] isEnabled: config.enabled=false 시 즉시 종료

## 비고

- SessionEnd는 Phase 1~5 기능의 통합 트리거 (AI 분석, 배치 임베딩, DB 정리)
- 비동기 작업은 모두 detached로 실행하여 훅 블로킹 방지
- 빈 세션도 요약 기록 (0 카운트, 빈 배열)
