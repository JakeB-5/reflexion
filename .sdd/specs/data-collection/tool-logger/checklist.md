# 체크리스트: tool-logger

## 스펙 완성도
- [x] 모든 REQ(201~206)에 GIVEN-WHEN-THEN 시나리오 포함
- [x] RFC 2119 키워드 적절히 사용 (SHALL 중심, SHOULD 크로스 도구 해결)
- [x] depends 필드 정확 (log-writer, error-kb)

## DESIGN.md 일치
- [x] REQ-DC-201: 도구 사용 기록 (type: 'tool_use', tool, meta, success) — DESIGN.md 4.4절 일치
- [x] REQ-DC-202: extractToolMeta 도구별 메타데이터 추출 (Bash 첫 단어, Read/Write/Edit 파일 경로, Task 에이전트 정보, Grep/Glob 패턴) — DESIGN.md 4.4절 구현 일치
- [x] REQ-DC-203: 동일 도구 해결 감지 (최근 50개 이벤트 조회, recordResolution 호출, 풍부한 컨텍스트) — DESIGN.md 4.4절 v7 P11 일치
- [x] REQ-DC-204: 크로스 도구 해결 감지 (Bash → Edit → Bash 패턴) — DESIGN.md 4.4절 일치
- [x] REQ-DC-205: Non-blocking 실행 (해결 감지 실패도 exit 0) — Constitution 2.1절 일치
- [x] REQ-DC-206: 시스템 활성화 체크 (isEnabled) — DESIGN.md 공통 패턴 일치

## 교차 참조
- [x] db.mjs insertEvent로 이벤트 기록
- [x] db.mjs queryEvents로 최근 50개 조회
- [x] error-kb.mjs recordResolution 함수 사용
- [x] session-summary에서 toolCounts, toolSequence 집계

## 테스트 계획
- [x] 일반 도구 사용 기록 (tool_use 이벤트)
- [x] Bash 명령어 첫 단어만 저장 (프라이버시)
- [x] Read/Write/Edit 파일 경로 저장
- [x] Task 도구 에이전트 정보 저장
- [x] Grep/Glob 패턴 저장
- [x] 알 수 없는 도구 빈 객체 반환
- [x] toolInput null/undefined 빈 객체 반환
- [x] 동일 도구 해결 감지 (Bash 에러 → Bash 성공)
- [x] 풍부한 해결 컨텍스트 (toolSequence 최대 5개, promptContext 200자)
- [x] 세션 스코프 제한 (다른 세션 에러 무시)
- [x] 크로스 도구 해결 감지 (Bash → Edit → Bash)
- [x] 이미 해결된 에러 무시 (helpingTools 목록 확인)
- [x] 해결 감지 중 오류 내부 try-catch
- [x] isEnabled false 시 즉시 exit 0

## 구현 주의사항
- **최근 50개 이벤트만 조회**: queryEvents({ sessionId, limit: 50 })로 O(n²) 방지
- **시간순 정렬 필수**: .sort((a, b) => new Date(a.ts) - new Date(b.ts)) 후 분석
- **recordResolution은 error-kb에서 import**: import { recordResolution } from '../lib/error-kb.mjs'
- **promptContext**: 해결 직전 마지막 프롬프트의 처음 200자 저장
- **toolSequence**: 에러와 성공 사이의 최대 5개 도구 사용 기록
- **해결 감지는 non-critical**: 실패해도 도구 사용 기록은 정상 완료
- **JSONL 제거**: 저장소는 SQLite events 테이블, insertEvent()/queryEvents() 사용
