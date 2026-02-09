---
feature: install-cli
created: 2026-02-09
status: draft
---

# 구현 계획: install-cli

> 시스템 설치/제거 CLI 스크립트 — Phase 1 작업 0번, 모든 훅 스크립트의 전제조건

---

## 개요

`bin/install.mjs`는 reflexion 시스템의 진입점이다. 디렉토리 구조 생성, 의존성 설치, 설정 초기화, 훅 등록을 자동화하여 수동 설정 15단계를 단일 명령어로 대체한다.

---

## 기술 결정

### 결정 1: Node.js 내장 모듈만 사용

**근거:** 설치 스크립트는 `npm install` 이전에 실행되므로 외부 패키지를 사용할 수 없다. `fs`, `path`, `child_process`, `os` 내장 모듈만으로 구현한다.

**대안 검토:**
- Shell script (bash) — 크로스 플랫폼 호환성 문제 (Windows)
- Python — 추가 런타임 의존성

### 결정 2: Read-Modify-Write 방식의 settings.json 수정

**근거:** `settings.json`에 기존 훅이 존재할 수 있으므로, 전체 파일을 읽고 병합 후 다시 쓰는 방식이 안전하다. 원자적 쓰기는 불필요하다 (설치는 사용자가 직접 실행하는 일회성 작업).

### 결정 3: 멱등성 우선 설계

**근거:** 사용자가 설치 스크립트를 여러 번 실행할 수 있으므로, 모든 단계에서 "이미 존재하면 건너뛰기" 패턴을 적용한다.

---

## 구현 단계

### Phase 1: 스캐폴드

CLI 진입점 파일 생성 및 인자 파싱

**산출물:**
- [ ] `~/.reflexion/bin/install.mjs` 파일 생성
- [ ] `process.argv` 파싱 (`--uninstall`, `--purge`)
- [ ] 상수 정의 (`SELF_GEN_DIR`, `SETTINGS_PATH`, `HOOK_EVENTS`)

### Phase 2: 설치 로직 구현

4단계 설치 프로세스 구현

**산출물:**
- [ ] 디렉토리 구조 생성 (5개 하위 디렉토리)
- [ ] `package.json` 생성 (존재 시 스킵)
- [ ] `npm install --production` 실행
- [ ] `config.json` 초기화 (존재 시 스킵)
- [ ] `settings.json` 훅 등록 (중복 방지)

### Phase 3: 제거 로직 구현

`--uninstall` 및 `--purge` 처리

**산출물:**
- [ ] 훅 선택적 제거 (`.reflexion` 경로 필터링)
- [ ] 빈 이벤트 배열 정리
- [ ] `--purge` 시 디렉토리 삭제

### Phase 4: 테스트

단위 테스트 및 통합 테스트

**산출물:**
- [ ] 멱등성 테스트 (반복 설치)
- [ ] 중복 등록 방지 테스트
- [ ] 제거 후 다른 훅 보존 테스트
- [ ] `--purge` 데이터 삭제 테스트

---

## 리스크 분석

| 리스크 | 영향도 | 완화 전략 |
|--------|--------|----------|
| `npm install` 실패 (네트워크/빌드) | HIGH | 에러 메시지 + exit 1, 재실행으로 복구 |
| `settings.json` 손상 | MEDIUM | JSON.parse 실패 시 빈 객체로 시작 |
| 권한 문제 (`~/.claude/` 접근) | LOW | `mkdirSync({ recursive: true })` 사용 |
| `better-sqlite3` 네이티브 빌드 실패 | MEDIUM | 에러 메시지에 빌드 도구 안내 포함 |

---

## 테스트 전략

### 단위 테스트

- 각 설치 단계별 독립 테스트
- 커버리지 목표: 80% 이상

### 통합 테스트

- 전체 설치 → 제거 → 재설치 사이클 테스트
- `settings.json` 병합 정확성 검증

---

## 다음 단계

1. [ ] 이 계획에 대한 검토 및 승인
2. [ ] 작업 분해 (tasks.md)
3. [ ] 구현 시작
