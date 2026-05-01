# redis-practice

Redis 운영 경험 기반의 개인 지식 베이스입니다. AWS EKS 환경에서 Bitnami Helm Chart로 운영하는 Redis 7.x Cluster/Sentinel 모드를 중심으로 작성되었습니다.

---

## 학습 경로 (Learning Path)

```
1단계: 기초
  └── docs/data-structures/string-hash.md       — String, Hash 자료구조
  └── docs/data-structures/list-set.md          — List, Set 자료구조
  └── docs/data-structures/sorted-set-stream.md — Sorted Set, Stream

2단계: 운영
  └── docs/operations/redis-cluster.md          — Redis Cluster 구성 및 운영
  └── docs/operations/redis-sentinel.md         — Redis Sentinel 고가용성
  └── docs/operations/persistence.md            — RDB/AOF 영속성 설정

3단계: 성능
  └── docs/performance/memory-management.md     — 메모리 관리, Eviction Policy
  └── docs/performance/connection-pooling.md    — 연결 풀 최적화
  └── docs/performance/slow-log-analysis.md     — Slow Log 분석 및 최적화

4단계: 보안/관측
  └── docs/security/auth-acl.md                 — AUTH 인증, ACL 사용자 관리
  └── docs/security/tls-encryption.md           — TLS 암호화 설정
  └── docs/observability/redis-metrics.md       — Prometheus 지표, Grafana
  └── docs/observability/redis-exporter.md      — Redis Exporter 설정
```

---

## 문서 목록

### data-structures/ — 자료구조

| 파일 | 주제 |
|------|------|
| `string-hash.md` | String, Hash — 기본 자료구조, EXPIRE, HSET/HGETALL |
| `list-set.md` | List, Set — LPUSH/RPOP, SADD/SMEMBERS, 사용 패턴 |
| `sorted-set-stream.md` | Sorted Set, Stream — 랭킹, 실시간 이벤트 스트림 |
| `data-type-selection.md` | 자료구조 선택 가이드 — 사용 사례별 최적 타입 |

### operations/ — 운영

| 파일 | 주제 |
|------|------|
| `redis-cluster.md` | Redis Cluster — 샤딩, 노드 추가/제거, Failover |
| `redis-sentinel.md` | Redis Sentinel — 고가용성, Failover, 클라이언트 설정 |
| `persistence.md` | 영속성 — RDB, AOF, AOF Rewrite, 복구 절차 |

### performance/ — 성능

| 파일 | 주제 |
|------|------|
| `memory-management.md` | 메모리 관리 — Eviction Policy, maxmemory, 메모리 최적화 |
| `connection-pooling.md` | 연결 풀 — Lettuce/Jedis 설정, 연결 수 최적화 |
| `slow-log-analysis.md` | Slow Log — SLOWLOG 분석, 명령어 최적화 |

### security/ — 보안

| 파일 | 주제 |
|------|------|
| `auth-acl.md` | 인증/인가 — AUTH, ACL 사용자, requirepass |
| `tls-encryption.md` | TLS 암호화 — 인증서 설정, 클라이언트 연결 |

### observability/ — 관측

| 파일 | 주제 |
|------|------|
| `redis-metrics.md` | Prometheus 지표 — 핵심 지표, Alert Rule |
| `redis-exporter.md` | Redis Exporter — 설치, ServiceMonitor, Grafana |

---

## 환경 정보

| 항목 | 값 |
|------|-----|
| Redis 버전 | 7.x |
| 배포 방식 | Bitnami Helm Chart (EKS) |
| Cluster 모드 | Redis Cluster (샤딩) |
| HA 모드 | Redis Sentinel |
| 네임스페이스 | `redis` |

---

## 커스텀 슬래시 커맨드

| 커맨드 | 설명 |
|--------|------|
| `/new-doc` | 신규 문서 스캐폴딩 |
| `/new-runbook` | 운영 Runbook 생성 |
| `/review-doc` | 문서 품질 검토 |
| `/add-troubleshooting` | 트러블슈팅 항목 추가 |
| `/search-kb` | 지식 베이스 검색 |
