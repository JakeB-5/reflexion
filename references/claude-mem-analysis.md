# claude-mem 프로젝트 분석

> 분석일: 2026-02-08
> 소스: https://github.com/thedotmack/claude-mem (v9.1.1)

## 한줄 요약

**Claude Code용 영속적 메모리 압축 시스템** — 세션 간 컨텍스트를 자동으로 보존하여 Claude가 이전 작업 이력을 기억하게 해주는 플러그인.

## 해결하는 문제

Claude Code는 세션이 끝나면 모든 컨텍스트를 잃는다. 새 세션을 시작하면 이전에 어떤 작업을 했는지, 어떤 버그를 고쳤는지, 프로젝트 구조가 어떤지 전혀 모른다. claude-mem은 이 **세션 간 기억 상실 문제**를 해결한다.

## 기술 스택

| 항목 | 기술 |
|------|------|
| 런타임 | Bun (Worker), Node.js >= 18 |
| 백엔드 | Express (port 37777) |
| DB | SQLite (`~/.claude-mem/claude-mem.db`) |
| 벡터 검색 | Chroma Vector DB |
| UI | React (Web Viewer) |
| 배포 | Claude Code Plugin Marketplace |
| 라이선스 | AGPL-3.0 |

## 핵심 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                    Claude Code Session                   │
│                                                         │
│  SessionStart ──→ UserPromptSubmit ──→ PostToolUse      │
│       │                  │                  │           │
│       ▼                  ▼                  ▼           │
│  ┌─────────┐      ┌──────────┐      ┌──────────────┐   │
│  │ Context │      │ Session  │      │ Observation  │   │
│  │ Inject  │      │  Init    │      │  Capture     │   │
│  └────┬────┘      └──────────┘      └──────┬───────┘   │
│       │                                     │           │
│       │              Stop ──→ Summarize     │           │
│       │                         │           │           │
└───────┼─────────────────────────┼───────────┼───────────┘
        │                         │           │
        ▼                         ▼           ▼
  ┌─────────────────────────────────────────────────┐
  │         Worker Service (Express, port 37777)     │
  │              Bun으로 실행                         │
  ├─────────────────────────────────────────────────┤
  │  SQLite DB          │  Chroma Vector DB          │
  │  (~/.claude-mem/    │  (시맨틱 검색)              │
  │   claude-mem.db)    │                            │
  └─────────────────────────────────────────────────┘
        ▲                         ▲
        │                         │
  ┌─────┴─────┐           ┌──────┴──────┐
  │ MCP Server│           │  Web Viewer │
  │ (5 tools) │           │  :37777 UI  │
  └───────────┘           └─────────────┘
```

## 동작 방식 (5단계 라이프사이클)

| 단계 | Hook 이벤트 | 동작 |
|------|------------|------|
| **1. 세션 시작** | `SessionStart` | 의존성 확인 → Worker 서비스 시작 → **이전 세션 컨텍스트를 Claude에 주입** |
| **2. 프롬프트 수신** | `UserPromptSubmit` | 세션 초기화, 현재 세션 메타데이터 기록 |
| **3. 도구 사용 관찰** | `PostToolUse` | 매 도구 사용 후 **observation(관찰)을 캡처**하여 DB에 저장 |
| **4. 세션 종료 전** | `Stop` | 축적된 관찰들을 **AI로 요약(summarize)** → 세션 완료 처리 |
| **5. 검색** | MCP Tools | 저장된 메모리를 시맨틱/키워드 검색으로 조회 |

### Hook 상세 구성 (hooks.json)

```
SessionStart:
  1. smart-install.js     (의존성 확인, timeout: 300s)
  2. worker-service start (Worker 시작, timeout: 60s)
  3. worker-service hook context (컨텍스트 주입, timeout: 60s)

UserPromptSubmit:
  1. worker-service start (Worker 확인)
  2. worker-service hook session-init (세션 초기화)

PostToolUse:
  1. worker-service start (Worker 확인)
  2. worker-service hook observation (관찰 캡처, timeout: 120s)

Stop:
  1. worker-service start (Worker 확인)
  2. worker-service hook summarize (요약 생성, timeout: 120s)
  3. worker-service hook session-complete (세션 완료, timeout: 30s)
```

## 핵심 컴포넌트

### 1. Hook 시스템 (데이터 수집)

- Claude Code의 **Hooks API**를 활용하여 모든 도구 사용을 자동 캡처
- 모든 hook은 `bun-runner.js` → `worker-service.cjs`를 통해 Worker에 위임
- `<private>` 태그로 민감한 내용은 저장에서 제외 가능
- Exit code 전략: 에러 시 exit 0 (Windows Terminal 탭 누적 방지)

### 2. Worker Service (백엔드)

- **Express** 기반 HTTP API (포트 37777)
- **Bun** 런타임으로 실행 및 프로세스 관리
- AI 처리(요약, 압축)는 비동기로 수행
- Web Viewer UI 제공 (실시간 메모리 스트림)

### 3. 저장소 (이중 DB)

- **SQLite** (`~/.claude-mem/claude-mem.db`) — 세션, 관찰, 요약 저장 + FTS5 검색
- **Chroma Vector DB** — 벡터 임베딩 기반 시맨틱 검색
- uv (Python 패키지 매니저)로 Chroma 관리

### 4. MCP Server (검색 인터페이스)

5개의 MCP 도구를 제공하며, **3-Layer 토큰 절약 패턴**을 사용:

```
search → 컴팩트 인덱스 (~50-100 토큰/결과)
  → timeline → 시간순 컨텍스트
    → get_observations → 필요한 것만 전체 상세 조회 (~500-1000 토큰/결과)
```

| MCP 도구 | 역할 |
|---------|------|
| `search` | 풀텍스트 검색, 타입/날짜/프로젝트 필터 |
| `timeline` | 특정 관찰 주변 시간순 컨텍스트 |
| `get_observations` | ID로 전체 상세 조회 |
| `save_memory` | 수동 메모리 저장 |
| `__IMPORTANT` | 워크플로우 문서 (항상 Claude에 노출) |

### 5. Progressive Disclosure (점진적 공개)

한 번에 모든 메모리를 주입하지 않고, **계층적으로 필요한 만큼만** 컨텍스트를 제공하여 토큰 비용을 최소화.

## 파일 시스템 구조

```
src/
├── hooks/           # Hook 응답 처리 (TypeScript → ESM 빌드)
├── services/
│   ├── sqlite/      # SQLite DB 관리
│   ├── sync/        # Chroma 벡터 동기화
│   └── worker-service.ts  # Express API 서버
├── servers/         # MCP 서버
├── sdk/             # Claude Agent SDK 연동
├── ui/viewer/       # React Web Viewer
├── utils/           # 유틸리티 (tag-stripping 등)
└── types/           # TypeScript 타입 정의

plugin/              # 빌드된 플러그인 (배포 대상)
├── hooks/hooks.json # Hook 설정
├── scripts/         # 실행 스크립트 (worker, mcp-server 등)
├── skills/          # mem-search 스킬
├── commands/        # CLI 명령어
├── modes/           # 실행 모드
└── ui/              # 빌드된 UI

~/.claude-mem/       # 런타임 데이터
├── claude-mem.db    # SQLite DB
├── chroma/          # Chroma Vector DB
├── settings.json    # 설정
└── logs/            # 로그
```

## self-generation 프로젝트와의 비교

| 관점 | claude-mem | self-generation |
|------|-----------|-----------------|
| **목적** | 세션 간 **메모리 보존** | 프롬프트 패턴 분석 → **자동 개선 제안** |
| **데이터 수집** | 도구 사용 관찰 캡처 | 프롬프트, 도구, 에러 수집 |
| **분석** | 관찰 → AI 요약/압축 | 패턴 감지 → 스킬/규칙/훅 생성 |
| **출력** | 이전 컨텍스트 주입 | 커스텀 스킬, CLAUDE.md 지시, 훅 워크플로우 |
| **저장** | SQLite + Chroma Vector | SQLite + sqlite-vec |
| **임베딩** | Chroma (외부 서비스, uv/Python) | Transformers.js (오프라인, 로컬) |
| **런타임** | Bun + Express | Node.js ES Modules |
| **배포 방식** | Claude Code Plugin Marketplace | 독립 설치 (`~/.self-generation/`) |
| **의존성** | 다수 (express, react, bun, uv 등) | 최소 3개 (better-sqlite3, sqlite-vec, transformers) |
| **관계** | 상호 보완적 — "기억하기" | 상호 보완적 — "패턴 발견 & 개선" |

## 기술적으로 주목할 점

1. **Plugin Marketplace 배포** — Claude Code의 공식 플러그인 시스템(`/plugin marketplace add`) 활용
2. **Bun 런타임** — Worker 프로세스 관리에 Node가 아닌 Bun 사용
3. **3-Layer 검색 패턴** — 토큰 효율을 위한 점진적 상세 조회 설계
4. **Exit Code 전략** — Hook 에러 시 exit 0으로 처리하여 Windows Terminal 탭 누적 방지
5. **Beta Channel** — Endless Mode 등 실험적 기능을 버전 스위칭으로 제공
6. **Privacy 태그** — `<private>` 태그로 민감 정보 저장 제외 (hook 레이어에서 edge processing)
7. **Pro 기능 분리** — 오픈소스 코어 + 유료 Pro 기능의 깔끔한 아키텍처 분리

## 참고할 수 있는 설계 패턴

- **Worker 위임 패턴**: 모든 Hook이 직접 처리하지 않고 Worker Service에 HTTP로 위임 → 비차단
- **Progressive Disclosure**: 메모리 검색 시 인덱스 → 타임라인 → 상세로 점진적 조회
- **Smart Install**: 의존성 캐시 체크로 불필요한 재설치 방지
- **MCP 통합**: 검색 기능을 MCP Server로 제공하여 Claude가 자연어로 메모리 검색 가능
