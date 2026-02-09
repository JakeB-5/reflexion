---
feature: install-cli
created: 2026-02-09
---

# 검증 체크리스트: install-cli

## 스펙 완전성

- [x] REQ-INF-001: 디렉토리 구조 생성 (5개 하위 디렉토리)
- [x] REQ-INF-002: package.json 생성 (3개 의존성, type: module)
- [x] REQ-INF-003: npm 의존성 설치 (production mode)
- [x] REQ-INF-004: config.json 초기화 (기본 설정값)
- [x] REQ-INF-005: settings.json 훅 등록 (8개 이벤트, 중복 방지)
- [x] REQ-INF-006: --uninstall 훅 해제 (선택적 제거)
- [x] REQ-INF-007: --uninstall --purge 완전 제거

## Constitution 준수

- [x] 비차단 훅: 설치 CLI 자체는 훅이 아니나, 등록하는 훅은 non-blocking 원칙 준수
- [x] 프라이버시: 로컬 전용 (`~/.reflexion/`)
- [x] 최소 의존성: Node.js 내장 모듈만 사용 (설치 스크립트 자체)
- [x] 명세 우선: 본 스펙 문서 작성 완료
- [x] RFC 2119: SHALL/SHOULD 키워드 사용

## DESIGN.md 일관성

- [x] 6.1.2절 install.mjs 코드와 일치
- [x] HOOK_EVENTS 매핑 8개 이벤트 일치
- [x] package.json 의존성 3개 일치
- [x] config.json 기본값 일치
- [x] settings.json hook group 형식 일치
- [x] Phase 1 로드맵 작업 0번 역할 반영

## GIVEN-WHEN-THEN 시나리오

- [x] REQ-INF-001: 2개 시나리오 (최초 생성, 멱등성)
- [x] REQ-INF-002: 2개 시나리오 (최초 생성, 존재 시 스킵)
- [x] REQ-INF-003: 2개 시나리오 (정상 설치, 실패 처리)
- [x] REQ-INF-004: 2개 시나리오 (최초 생성, 존재 시 스킵)
- [x] REQ-INF-005: 3개 시나리오 (최초 등록, 병합, 중복 방지)
- [x] REQ-INF-006: 3개 시나리오 (선택적 제거, 미존재, 데이터 보존)
- [x] REQ-INF-007: 2개 시나리오 (완전 제거, purge 단독 사용)
