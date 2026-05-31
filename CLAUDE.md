# CLAUDE.md — redis-practice 지식 베이스

Redis 운영 경험 기반의 개인 지식 베이스입니다. 문서 추가/수정 시 아래 가이드를 따릅니다.

## 프로젝트 설정

- **환경**: EKS
- **Redis 버전**: 7.x
- **배포 방식**: Bitnami Helm Chart
- **Cluster 모드**: Redis Cluster (샤딩) / Redis Sentinel (HA)
- **네임스페이스**: `redis`
- **접속 엔드포인트**: `redis-master.redis.svc.cluster.local:6379`

---

## 프로젝트 구조

```
redis-practice/
├── docs/
│   ├── README.md                       # 문서 구조 안내
│   ├── data-structures/                # 자료구조별 사용법 및 패턴
│   ├── operations/                     # Cluster/Sentinel 운영, 영속성
│   ├── performance/                    # 메모리, 연결 풀, Slow Log, 네트워크 대역폭
│   ├── security/                       # 인증/인가, TLS
│   ├── observability/                  # 지표 수집, Grafana
│   ├── templates/                      # 재사용 문서 템플릿
│   ├── rules/                          # Claude 작성 규칙
│   └── agents/                         # Claude 전문 에이전트
├── ops/
│   └── config/                         # Redis 설정 예제
├── CLAUDE.md                           # Claude 작업 기준
├── AGENTS.md                           # Codex 작업 기준
└── .gitignore                          # 로컬/민감 파일 제외 규칙
```

`.claude/`는 Claude Code 로컬 워크스페이스입니다. 개인 설정, 임시 커맨드, 로컬 세션 정보가 섞일 수 있으므로 Git에 커밋하지 않습니다.

---

## 커스텀 슬래시 커맨드

| 커맨드 | 사용법 | 설명 |
|--------|--------|------|
| `/new-doc` | `/new-doc performance pipeline-optimization` | 신규 문서 스캐폴딩 |
| `/new-runbook` | `/new-runbook operations failover-recovery` | 운영 Runbook 생성 |
| `/review-doc` | `/review-doc docs/security/auth-acl.md` | 문서 품질 검토 |
| `/add-troubleshooting` | `/add-troubleshooting docs/operations/redis-cluster.md <증상>` | 트러블슈팅 추가 |
| `/search-kb` | `/search-kb eviction memory` | 지식 베이스 키워드 검색 |

---

## 파일 네이밍 규칙

```
docs/{카테고리}/{주제}.md
```

- 카테고리: `data-structures`, `operations`, `performance`, `security`, `observability`
- 주제: 소문자 영어, 하이픈 구분
- 예시: `docs/performance/pipeline-optimization.md`, `docs/operations/backup-restore.md`

---

## 문서 작성 원칙

1. **실제 경험 기반** — 운영 중 실제로 겪은 이슈와 해결 방법 위주
2. **재현 가능한 코드** — redis-cli 명령어 복붙 즉시 적용 가능
3. **원인 중심 트러블슈팅** — 증상만 나열하지 말고 근본 원인 설명
4. **한국어 기술 문서** — 주요 개념은 영어 원문 병기
5. **모니터링 필수** — 모든 문서에 Prometheus 지표 또는 진단 명령어 포함

세부 규칙은 `docs/rules/` 디렉토리를 참조합니다.

---

## 카테고리별 문서 목록

### docs/data-structures/
| 파일 | 주제 |
|------|------|
| `string-hash.md` | String, Hash 자료구조 |
| `list-set.md` | List, Set 자료구조 |
| `sorted-set-stream.md` | Sorted Set, Stream 자료구조 |
| `data-type-selection.md` | 자료구조 선택 가이드 |

### docs/operations/
| 파일 | 주제 |
|------|------|
| `redis-cluster.md` | Redis Cluster 구성 및 운영 |
| `redis-sentinel.md` | Redis Sentinel 고가용성 |
| `persistence.md` | RDB/AOF 영속성 |

### docs/performance/
| 파일 | 주제 |
|------|------|
| `memory-management.md` | 메모리 관리 및 Eviction |
| `connection-pooling.md` | 연결 풀 최적화 |
| `slow-log-analysis.md` | Slow Log 분석 |
| `network-bandwidth-scaling.md` | 네트워크 대역폭 병목과 스케일링 안티패턴 |

### docs/security/
| 파일 | 주제 |
|------|------|
| `auth-acl.md` | AUTH 인증 및 ACL |
| `tls-encryption.md` | TLS 암호화 |

### docs/observability/
| 파일 | 주제 |
|------|------|
| `redis-metrics.md` | Prometheus 지표 |
| `redis-exporter.md` | Redis Exporter 설정 |

---

## 추가 예정 주제 (백로그)

- `docs/performance/pipeline-optimization.md` — Pipeline, MULTI/EXEC, Lua Script
- `docs/operations/backup-restore.md` — RDB/AOF 백업 및 복구 절차
- `docs/data-structures/pub-sub.md` — Pub/Sub 패턴
- `docs/operations/redis-upgrade.md` — Redis 버전 업그레이드 전략
