# 체크리스트: batch-embeddings

## 스펙 완성도
- [ ] 모든 REQ에 GIVEN-WHEN-THEN 시나리오 포함
- [ ] RFC 2119 키워드 적절히 사용 (SHALL, SHOULD, MAY)
- [ ] depends 필드 정확 (`realtime-assist/embedding-daemon, realtime-assist/error-kb, data-collection/log-writer`)

## DESIGN.md 일치
- [ ] REQ-RA-601: 10초 시작 지연 (`setTimeout(r, 10000)`)이 DESIGN.md 9.2.4와 일치
- [ ] REQ-RA-602: `busy_timeout = 10000` 설정이 DESIGN.md 9.2.4와 일치
- [ ] REQ-RA-603: 임베딩 데몬 대기 루프 (15회, 1초 간격, `isServerRunning()` + `startServer()`)가 DESIGN.md와 일치
- [ ] REQ-RA-604: 미임베딩 에러 KB 조회 SQL (`NOT IN (SELECT error_kb_id FROM vec_error_kb)`)이 DESIGN.md와 일치
- [ ] REQ-RA-604: DELETE+INSERT 패턴 (vec_error_kb)이 DESIGN.md와 일치
- [ ] REQ-RA-604: `Buffer.from(new Float32Array(embedding).buffer)` 변환이 DESIGN.md와 일치
- [ ] REQ-RA-605: `loadSkills(projectPath)` 전역+프로젝트 스킬 로드가 DESIGN.md와 일치
- [ ] REQ-RA-605: `INSERT OR REPLACE INTO skill_embeddings` UPSERT 패턴이 DESIGN.md와 일치
- [ ] REQ-RA-605: `extractPatterns(skill.content)` 키워드 추출이 DESIGN.md 9.3.4와 일치
- [ ] REQ-RA-605: 스킬 콘텐츠 500자 제한 (`skill.content.slice(0, 500)`)이 DESIGN.md와 일치
- [ ] REQ-RA-606: 최상위 try-catch + `process.exit(0)` 종료 보장이 DESIGN.md 훅 원칙과 일치

## 크로스 레퍼런스
- [ ] session-summary (REQ-DC-405)에서 detached spawn으로 실행되는 호출부 확인
- [ ] embedding-daemon (REQ-RA-501~510)의 `isServerRunning()`, `startServer()` API 호환
- [ ] error-kb (REQ-RA-001~005)의 `error_kb` 테이블 스키마와 벡터 테이블 일치
- [ ] skill-matcher (REQ-RA-101~104)의 `loadSkills()`, `extractPatterns()` API 호환
- [ ] log-writer (REQ-DC-005)의 `getDb()`, `generateEmbeddings()` API 호환

## 테스트 계획
- [ ] 10초 지연 후 정상 실행 시나리오
- [ ] 임베딩 데몬 미실행 → 시작 → 대기 성공 시나리오
- [ ] 임베딩 데몬 15초 타임아웃 시나리오
- [ ] 미임베딩 에러 KB 엔트리 배치 처리 시나리오
- [ ] 미임베딩 엔트리 없는 경우 스킵 시나리오
- [ ] 부분 임베딩 실패 (null 반환) 시 건너뛰기 시나리오
- [ ] 스킬 UPSERT + 벡터 갱신 시나리오
- [ ] lastInsertRowid 폴백 (SELECT id) 시나리오
- [ ] 전체 예외 시 exit 0 보장 시나리오
- [ ] WAL + busy_timeout 동시 쓰기 경합 시나리오
