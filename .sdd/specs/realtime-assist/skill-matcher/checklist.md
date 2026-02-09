---
feature: skill-matcher
created: 2026-02-09
---

# 검증 체크리스트: skill-matcher

## 스펙 완전성

- [x] REQ-RA-101: 스킬 목록 로드 (loadSkills, 전역+프로젝트 스캔)
- [x] REQ-RA-102: 프롬프트-스킬 매칭 (matchSkill, 벡터 우선 + 키워드 fallback)
- [x] REQ-RA-103: 스킬 임베딩 갱신 (refreshSkillEmbeddings, mtime 기반)
- [x] REQ-RA-104: 패턴 추출 (extractPatterns, "감지된 패턴" 섹션 파싱)

## Constitution 준수

- [x] 비차단 훅: 유틸리티 모듈이므로 N/A
- [x] 프라이버시: 로컬 전용 (skill_embeddings 테이블)
- [x] 최소 의존성: better-sqlite3 + sqlite-vec + Transformers.js (db.mjs 통해 간접 사용)
- [x] 명세 우선: 본 스펙 문서 작성 완료
- [x] RFC 2119: SHALL/SHOULD 키워드 사용

## DESIGN.md 일관성

- [x] skill_embeddings + vec_skill_embeddings 테이블 스키마 일치
- [x] 벡터 유사도 임계값 0.76 (distance < 0.76 = 매치)
- [x] 키워드 매칭 임계값 50% (fallback)
- [x] Transformers.js `paraphrase-multilingual-MiniLM-L12-v2` 모델 (384차원)
- [x] 시노님맵 제거 (v8 변경사항)
- [x] generateEmbeddings() 유틸리티 사용 (db.mjs)
- [x] Phase 5 실시간 지원 역할 반영

## GIVEN-WHEN-THEN 시나리오

- [x] REQ-RA-101: 3개 시나리오 (전역+프로젝트 로드, 디렉토리 부재, 전역만 로드)
- [x] REQ-RA-102: 5개 시나리오 (벡터 매칭, 교차 언어, 키워드 fallback, 임계값 미달, 벡터 실패 폴백)
- [x] REQ-RA-103: 3개 시나리오 (새 스킬 임베딩, 변경 감지, 최신 상태)
- [x] REQ-RA-104: 2개 시나리오 (패턴 추출, 패턴 없음)

## 의존성 검증

- [x] data-collection/log-writer: generateEmbeddings(), vectorSearch(), getDb() 함수 존재 확인

## 구현 시 주의사항

### 테이블 스키마
```sql
-- 메타데이터 테이블
CREATE TABLE skill_embeddings (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL UNIQUE,
  source_path TEXT NOT NULL,
  description TEXT,
  keywords TEXT,  -- JSON array
  updated_at TEXT NOT NULL
);

-- 벡터 가상 테이블
CREATE VIRTUAL TABLE vec_skill_embeddings USING vec0(
  skill_id INTEGER PRIMARY KEY,
  embedding float[384]
);
```

### loadSkills() 반환 형식
```javascript
[
  {
    name: string,          // 파일명 (.md 제거)
    scope: 'global' | 'project',
    content: string,       // 파일 전체 내용
    description: string | null,  // 첫 비어있지 않고 # 없는 줄
    sourcePath: string     // 전체 경로
  }
]
```

### matchSkill() 반환 형식
```javascript
{
  name: string,
  match: 'vector' | 'keyword',
  confidence: number,  // 0~1 (vector: 1-distance, keyword: matchCount/total)
  scope: 'global' | 'project'
}
```

### 매칭 우선순위 (2단계)
1. **벡터 유사도 검색 (primary)**:
   - `generateEmbeddings([prompt])` → 프롬프트 임베딩 생성
   - `vectorSearch('skill_embeddings', 'vec_skill_embeddings', embedding, 1)` → 가장 유사한 스킬 검색
   - `distance < 0.76` → 매치 판정
   - 실패 시 예외 흡수하고 키워드 매칭으로 폴스루

2. **키워드 패턴 매칭 (fallback)**:
   - `extractPatterns(content)` → 패턴 키워드 추출
   - 50% 이상 매칭 시 반환
   - 소문자 비교 (`toLowerCase()`)
   - 3자 이상 단어만 대상

### extractPatterns() 로직
1. "감지된 패턴" 헤딩 찾기
2. 다음 `#` 헤딩 전까지 `- ` 접두사 항목 추출
3. `- ` 제거 + 양끝 따옴표 제거
4. 섹션 없으면 빈 배열 반환

### refreshSkillEmbeddings() 로직
1. `skill_embeddings`에서 임베딩 없거나 `updated_at < file.mtime`인 엔트리 조회
2. description + keywords로 `generateEmbeddings()` 호출 (async)
3. `vec_skill_embeddings`에 INSERT OR REPLACE
4. `skill_embeddings.updated_at` 갱신
5. 실패 시 해당 스킬 건너뛰고 계속 처리

### 에러 처리 정책
- 모든 함수: 예외 발생 시 `null` 또는 빈 값 반환 (프로세스 중단 금지)
- loadSkills(): 디렉토리 부재 시 빈 배열 반환
- matchSkill(): 벡터 검색 실패 시 키워드 폴백, 모두 실패 시 `null`
- refreshSkillEmbeddings(): 임베딩 생성 실패 시 해당 스킬 건너뛰기

### 벡터 검색 임계값
- **cosine distance < 0.76** → 매치
- confidence = 1 - distance (0~1 범위)

### 키워드 매칭 임계값
- **50% 이상** → 매치
- confidence = matchCount / patternWords.length

### 교차 언어 매칭
- 한국어 프롬프트 → 영어 스킬 매칭 가능
- 영어 프롬프트 → 한국어 스킬 매칭 가능
- `paraphrase-multilingual-MiniLM-L12-v2` 모델의 다국어 지원 활용

### 시노님맵 제거 (v8 변경)
- 기존 `loadSynonymMap()` 함수 제거됨
- 벡터 유사도 검색이 의미적 유사성을 네이티브하게 처리
- 동의어 목록 수동 관리 불필요

### 호출 시점
- `loadSkills()`: prompt-logger 훅에서 즉시 호출
- `matchSkill()`: prompt-logger 훅에서 즉시 호출 (20ms 이내)
- `refreshSkillEmbeddings()`: SessionEnd 또는 SessionStart 시점 (비동기, 성능 영향 없음)

### 성능 요구사항
- `matchSkill()`: 20ms 이내 (SHOULD)
- 벡터 검색: 스킬 100개 이상에서도 성능 저하 없음 (sqlite-vec 인덱스 활용)
- `refreshSkillEmbeddings()`: 세션 경계에만 수행 (실시간 성능 영향 없음)
