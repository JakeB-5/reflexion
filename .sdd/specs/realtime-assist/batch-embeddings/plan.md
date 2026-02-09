---
feature: batch-embeddings
created: 2026-02-09
status: draft
---

# 구현 계획: batch-embeddings

> 배치 임베딩 프로세서 — SessionEnd 후 비동기 벡터 임베딩 생성

---

## 개요

`lib/batch-embeddings.mjs`는 `session-summary.mjs`에서 detached spawn으로 실행되는 독립 스크립트이다. 에러 KB의 미임베딩 엔트리와 스킬 파일에 대해 벡터 임베딩을 생성하여 sqlite-vec 가상 테이블에 저장한다.

---

## 기술 결정

### 결정 1: 10초 시작 지연

**근거:** session-summary와 ai-analyzer가 동시에 DB에 쓰기하므로 busy retry 빈도를 줄이기 위한 최적화. 경합 방지를 보장하지는 않으며, WAL + busy_timeout이 최종 안전장치.

**대안 검토:**
- 지연 없이 즉시 실행 — busy retry 빈도가 높아짐
- 잠금 파일 기반 조정 — 구현 복잡도 대비 이득이 적음

### 결정 2: 임베딩 데몬 재사용

**근거:** Transformers.js 모델 로딩이 1-4초 소요되므로 매번 로딩하는 대신 상주 데몬과 Unix socket으로 통신하여 ~5ms/건 달성.

**대안 검토:**
- 스크립트 내 직접 모델 로딩 — 1-4초 초기화 오버헤드
- HTTP API — Unix socket 대비 불필요한 네트워크 오버헤드

### 결정 3: DELETE + INSERT 패턴 (vec 테이블)

**근거:** sqlite-vec 가상 테이블은 UPDATE를 지원하지 않을 수 있으므로 DELETE 후 INSERT로 벡터를 교체.

**대안 검토:**
- UPDATE — sqlite-vec 가상 테이블 호환성 불확실
- UPSERT — vec0 가상 테이블에서 ON CONFLICT 미지원

---

## 구현 단계

### Phase 1: 스캐폴드

스크립트 파일 및 의존성 설정

**산출물:**
- [ ] `~/.self-generation/lib/batch-embeddings.mjs` 파일 생성
- [ ] import 설정: `db.mjs`, `skill-matcher.mjs`, `embedding-client.mjs`
- [ ] `process.argv[2]` 인자 파싱
- [ ] 최상위 try-catch + `process.exit(0)` 래퍼

### Phase 2: 임베딩 데몬 대기

데몬 상태 확인 및 시작 대기 루프

**산출물:**
- [ ] 10초 시작 지연 (`setTimeout`)
- [ ] `isServerRunning()` 확인
- [ ] 미실행 시 `startServer()` 호출
- [ ] 15회 × 1초 간격 대기 루프

### Phase 3: 에러 KB 배치 임베딩

미임베딩 에러 KB 엔트리 처리

**산출물:**
- [ ] `busy_timeout = 10000` 설정
- [ ] 미임베딩 엔트리 조회 쿼리
- [ ] `generateEmbeddings()` 호출
- [ ] `Float32Array` → `Buffer` 변환
- [ ] `DELETE` + `INSERT` 벡터 저장

### Phase 4: 스킬 임베딩 갱신

스킬 메타데이터 UPSERT 및 벡터 생성

**산출물:**
- [ ] `loadSkills(projectPath)` 호출
- [ ] `skill_embeddings` 테이블 UPSERT
- [ ] `extractPatterns()` → `keywords` JSON 생성
- [ ] `skillId` 획득 (lastInsertRowid 또는 SELECT)
- [ ] 스킬 콘텐츠 500자 임베딩 생성
- [ ] `vec_skill_embeddings` 벡터 저장

### Phase 5: 테스트

배치 처리 검증

**산출물:**
- [ ] 에러 KB 미임베딩 엔트리 처리 검증
- [ ] 스킬 임베딩 UPSERT 검증
- [ ] 데몬 미실행 시 graceful 종료 검증
- [ ] DB 접근 실패 시 exit 0 검증
- [ ] busy_timeout 동작 검증

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| 임베딩 데몬 시작 실패 | MEDIUM | 15초 대기 후 graceful 종료 (exit 0) |
| DB 쓰기 경합 (WAL contention) | MEDIUM | 10초 지연 + busy_timeout 10초 |
| 대량 미임베딩 엔트리 | LOW | 순차 처리, 개별 실패 시 건너뛰기 |
| 스킬 파일 읽기 실패 | LOW | `loadSkills()` 내부에서 처리 |
| 프로세스 비정상 종료 | LOW | 최상위 try-catch, exit 0 보장 |

---

## 테스트 전략

### 단위 테스트

- 미임베딩 에러 KB 조회 쿼리 검증
- Float32Array → Buffer 변환 정확성
- skillId 획득 로직 (lastInsertRowid / SELECT 폴백)
- 커버리지 목표: 80% 이상

### 통합 테스트

- DB에 에러 KB 엔트리 삽입 → 배치 임베딩 → vec_error_kb 확인
- 스킬 파일 생성 → 배치 임베딩 → skill_embeddings + vec_skill_embeddings 확인
- 데몬 미실행 상태에서 전체 플로우 (graceful 종료)

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] 작업 분해 (tasks.md)
3. [ ] 구현 시작
