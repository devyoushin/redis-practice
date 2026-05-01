# Redis 코드 및 설정 규칙

문서 내 Redis CLI 명령어, 설정 파일, Helm values 코드 작성 시 아래 규칙을 따릅니다.

---

## redis-cli 명령어 규칙

### 기본 형식
```redis-cli
# 인증이 필요한 경우
AUTH <REDIS_PASSWORD>

# 또는 -a 옵션
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <REDIS_PASSWORD>
```

### Kubernetes 환경 명령어
```bash
# Pod exec를 통한 redis-cli 실행
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a $(kubectl get secret redis -n redis -o jsonpath='{.data.redis-password}' | base64 -d)

# 포트 포워딩 후 로컬 접속
kubectl port-forward -n redis svc/redis-master 6379:6379
redis-cli -h localhost -p 6379 -a <REDIS_PASSWORD>
```

### 플레이스홀더 형식
- `<REDIS_PASSWORD>` — Redis 비밀번호
- `<HOST>` — Redis 호스트
- `<KEY>` — Redis 키 이름
- `<VALUE>` — Redis 값

---

## redis.conf 설정 규칙

### 형식
- 설정 값에 단위 주석 포함
- 운영 환경 권장값 명시

```
# 메모리 설정
maxmemory 4gb                    # 최대 메모리 4GB (Pod 메모리의 70~80%)
maxmemory-policy allkeys-lru     # 전체 키 LRU Eviction

# 영속성 설정
save 900 1                       # 900초 내 1개 이상 변경 시 RDB 저장
save 300 10                      # 300초 내 10개 이상 변경 시 RDB 저장
appendonly yes                   # AOF 활성화
appendfsync everysec             # 1초마다 fsync (성능/안정성 균형)
```

---

## Bitnami Helm Values 규칙

### 파일 헤더
```yaml
# Bitnami Redis Helm Chart values
# Chart: bitnami/redis
# Version: 18.x
# Redis Version: 7.x
```

### 필수 포함 항목
```yaml
# 인증 설정
auth:
  enabled: true
  password: ""                   # Kubernetes Secret으로 관리 권장

# 리소스 설정 (반드시 requests/limits 명시)
master:
  resources:
    requests:
      memory: 256Mi
      cpu: "100m"
    limits:
      memory: 512Mi
      cpu: "500m"
```

---

## INFO 명령어 섹션

자주 사용하는 INFO 섹션:

| 섹션 | 명령어 | 주요 확인 항목 |
|------|--------|-------------|
| 메모리 | `INFO memory` | `used_memory_human`, `maxmemory_human`, `mem_fragmentation_ratio` |
| 복제 | `INFO replication` | `role`, `connected_slaves`, `master_link_status` |
| 통계 | `INFO stats` | `total_commands_processed`, `keyspace_hits`, `keyspace_misses` |
| 클라이언트 | `INFO clients` | `connected_clients`, `blocked_clients` |
| 영속성 | `INFO persistence` | `rdb_last_bgsave_status`, `aof_enabled` |
| 클러스터 | `INFO cluster` | `cluster_enabled`, `cluster_state` |

---

## 명령어 보안 규칙

- `FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG`, `SLAVEOF` 등 위험 명령어는 항상 주의 문구 포함
- 예시:
```redis-cli
# ⚠️ 주의: 모든 데이터 삭제 — 운영 환경 실행 금지
FLUSHALL
```

- 비밀번호는 직접 기재하지 않고 환경 변수 또는 Secret 참조
