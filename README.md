# redis-practice

Redis를 systemd, Docker Compose, 패키지/tar, Kubernetes/Helm 관점에서 학습하고 운영 절차를 정리하는 개인 학습 공간입니다.

## 어디서 시작할까

- 문서 지도: `docs/README.md`
- 설치 방식 선택: `docs/01-installation/README.md`
- 첫 문서: `docs/02-data-structures/string-hash.md`
- 운영 보조 자료: `ops/README.md`
- AI 작업 지침: `CLAUDE.md`, `AGENTS.md -> CLAUDE.md`

## 구조

| 경로 | 내용 |
|------|------|
| `docs/` | Redis 설치, 자료구조, 운영, 성능, 보안, 관측 문서 |
| `docs/90-standards/` | 문서 작성 및 운영 규칙 |
| `docs/91-templates/` | 재사용 문서 템플릿 |
| `docs/99-agents/` | Claude 에이전트 프롬프트 |
| `ops/` | redis.conf, systemd unit, Docker Compose, Helm values 예시 |
| `CLAUDE.md` | Claude/Codex 공통 작업 지침 원본 |
| `AGENTS.md -> CLAUDE.md` | Codex/agent 작업 지침 링크 |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | VM/systemd, Docker Compose, Kubernetes |
| Redis | 7.x |
| 배포 | systemd 우선, Docker Compose는 개발/검증, Helm은 Kubernetes 운영 역량이 있을 때 |
