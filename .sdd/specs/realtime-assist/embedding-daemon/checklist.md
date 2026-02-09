---
spec: embedding-daemon
created: 2026-02-09
---

# Checklist: embedding-daemon

## 스펙 완성도

- [x] REQ-RA-501: Unix Socket 서버 생성 — 소켓 경로, stale 파일 정리
- [x] REQ-RA-502: Transformers.js 모델 로딩 — 모델명, 캐시 디렉토리, 384차원
- [x] REQ-RA-503: Newline-Delimited JSON 프로토콜 — embed/health/unknown/잘못된 JSON
- [x] REQ-RA-504: Idle Timeout 자동 종료 — 30분, 타이머 리셋, 초기 설정
- [x] REQ-RA-505: EADDRINUSE 처리 — 중복 인스턴스 방지, 기타 에러 throw
- [x] REQ-RA-506: Graceful Shutdown — SIGTERM, SIGINT, 클라이언트 에러 무시
- [x] REQ-RA-507: embedViaServer 인터페이스 — 정상 요청, auto-start, 재시도 실패
- [x] REQ-RA-508: 소켓 통신 및 타임아웃 — 10초 타임아웃, embeddings 필드 검증
- [x] REQ-RA-509: isServerRunning 상태 확인 — health check, 500ms 타임아웃
- [x] REQ-RA-510: startServer 데몬 시작 — detached spawn, unref

## DESIGN.md 일치 검증

- [x] 서버 소켓 경로: `/tmp/self-gen-embed.sock`
- [x] 모델명: `Xenova/paraphrase-multilingual-MiniLM-L12-v2`
- [x] 모델 캐시: `~/.reflexion/models/`
- [x] idle timeout: 30분 (1,800,000ms)
- [x] 클라이언트 타임아웃: 10초 (10,000ms)
- [x] health check 타임아웃: 500ms
- [x] auto-start 대기 시간: 5초 (5,000ms)
- [x] detached spawn + unref
- [x] EADDRINUSE → process.exit(0)
- [x] SIGTERM/SIGINT → server.close() + process.exit(0)
- [x] stale 소켓: existsSync + unlinkSync
- [x] 프로토콜: newline-delimited JSON

## 교차 참조 검증

- [x] `data-collection/log-writer` REQ-DC-008: `generateEmbeddings()`가 `embedViaServer()`를 사용
- [x] `ai-analysis/session-start-hook` REQ-SSH-006: SessionStart에서 `isServerRunning()` + `startServer()` 호출
- [x] config.json `embedding.server` 섹션: socketPath, idleTimeoutMinutes, clientTimeoutMs 설정 가능

## SDD 규칙 준수

- [x] RFC 2119 키워드 사용 (SHALL, SHOULD, MAY)
- [x] 모든 요구사항에 GIVEN-WHEN-THEN 시나리오 포함
- [x] REQ 번호 체계 준수 (REQ-RA-5xx)
- [x] 문서 언어: 한글 / 코드·함수명: 영어
- [x] constitution_version 명시
