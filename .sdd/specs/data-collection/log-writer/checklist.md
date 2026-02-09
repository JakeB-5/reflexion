# 체크리스트: log-writer

## 스펙 완성도
- [x] 모든 REQ(001~012)에 GIVEN-WHEN-THEN 시나리오 포함
- [x] RFC 2119 키워드 적절히 사용 (SHALL 중심)
- [x] depends 필드 정확 (3개 npm 패키지)

## DESIGN.md 일치
- [x] REQ-DC-001: events 테이블 INSERT-only, UPDATE 트리거 금지 — DESIGN.md 3.1.1 스키마 일치
- [x] REQ-DC-002: queryEvents 필터 조건 (type, sessionId, project, projectPath, since, search, limit) — DESIGN.md 3.2 함수 시그니처 일치
- [x] REQ-DC-003: initDb 테이블 생성 (events, error_kb, feedback, analysis_cache, skill_embeddings) + FTS5/vec0 가상 테이블 — DESIGN.md 3.1 스키마 일치
- [x] REQ-DC-004: pruneOldEvents 보관 기한 초과 DELETE — DESIGN.md 3.2 유틸리티 함수 일치
- [x] REQ-DC-005: loadConfig 설정 필드 (enabled, collectPromptText, retentionDays, dbPath, analysisOnSessionEnd, analysisDays, analysisCacheMaxAgeHours, embedding) — DESIGN.md 9.6 config.json 일치
- [x] REQ-DC-006: readStdin Promise 기반 stdin 파싱, 5초 타임아웃 — DESIGN.md 3.2 구현 일치
- [x] REQ-DC-007: getDb 싱글톤 DB 연결, WAL 모드, sqlite-vec 로드, migrateV9 호출 — DESIGN.md 3.2 초기화 로직 일치
- [x] REQ-DC-008: generateEmbeddings 임베딩 데몬 호출, 빈 배열 폴백 — DESIGN.md 3.2.12 구현 일치
- [x] REQ-DC-009: vectorSearch VEC_TABLE_REGISTRY, 2단계 검색 — DESIGN.md 3.2.13 구현 일치
- [x] REQ-DC-010: getSessionEvents 위임 호출 — DESIGN.md 3.2 유틸리티 함수 일치
- [x] REQ-DC-011: stripPrivateTags <private> 태그 제거 — DESIGN.md 3.2.11 구현 일치
- [x] REQ-DC-012: getProjectName/getProjectPath CLAUDE_PROJECT_DIR 환경변수 우선 — DESIGN.md 3.2 유틸리티 함수 일치

## 교차 참조
- [x] data-collection 5개 훅 모두 db.mjs를 의존성으로 명시
- [x] ai-analysis/ai-analyzer가 queryEvents 사용
- [x] suggestion-engine/feedback-tracker가 insertEvent 사용
- [x] realtime-assist 5개 스펙이 generateEmbeddings/vectorSearch 사용
- [x] VEC_TABLE_REGISTRY 구조가 error-kb, skill-matcher와 일치

## 테스트 계획
- [x] insertEvent 성공 시나리오 (공통 필드 + data JSON 분리)
- [x] insertEvent 자동 DB 초기화 (파일 미존재)
- [x] queryEvents 다중 필터 조합 (type + sessionId + limit)
- [x] queryEvents FTS5 전문 검색 (search 파라미터)
- [x] initDb 멱등성 (IF NOT EXISTS)
- [x] initDb FTS5 트리거 생성 (INSERT/DELETE 동기화, UPDATE 금지)
- [x] initDb vec0 가상 테이블 생성 실패 무시 (try-catch)
- [x] pruneOldEvents 보관 기한 초과 삭제 (events + error_kb)
- [x] loadConfig 파일 미존재 시 빈 객체 반환
- [x] isEnabled 기본값 true
- [x] readStdin 타임아웃 5초
- [x] getDb 싱글톤 재호출 시 기존 연결 반환
- [x] generateEmbeddings 데몬 미실행 시 빈 배열 반환
- [x] vectorSearch 미등록 테이블 빈 배열 반환
- [x] stripPrivateTags 닫히지 않은 태그 무시
- [x] getProjectPath CLAUDE_PROJECT_DIR 우선 사용

## 구현 주의사항
- **JSONL 제거 완료**: rotateIfNeeded, getLogFile 함수는 구현하지 말 것 (SQLite 전환)
- **WAL 모드 필수**: 훅 간 동시 쓰기 충돌 방지
- **migrateV9 자동 실행**: getDb 초기화 시 input_hash 컬럼과 events_fts 가상 테이블 추가
- **VEC_TABLE_REGISTRY 확장성**: 새 벡터 테이블 추가 시 하드코딩 없이 등록만으로 동작
- **FTS5 동기화 트리거**: events 테이블 INSERT/DELETE 시 events_fts 자동 업데이트, UPDATE 트리거로 금지
