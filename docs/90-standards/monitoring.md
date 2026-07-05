# 모니터링 및 확인 섹션 작성 규칙

모든 문서의 `## 4. 모니터링 및 확인` 섹션은 아래 형식을 따릅니다.

---

## 필수 포함 항목

### 1. redis-cli 진단 명령어

문서 주제와 관련된 `redis-cli` 진단 명령어를 포함합니다.

```bash
# 예시: 메모리 관련 문서
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO memory

# 예시: 복제 상태 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO replication
```

### 2. Prometheus 지표 테이블

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_memory_used_bytes` | 사용 중인 메모리 | `maxmemory`의 85% 초과 |
| `redis_connected_clients` | 연결된 클라이언트 수 | `maxclients`의 80% 초과 |

### 3. Grafana 쿼리 예시 (선택)

복잡한 지표는 PromQL 쿼리를 포함합니다.

```promql
# 메모리 사용률 (%)
redis_memory_used_bytes / redis_config_maxmemory * 100
```

---

## 지표 테이블 형식

```markdown
| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `지표명` | 한국어 설명 | 수치 기준 또는 조건 |
```

- **알람 기준**: 가능하면 수치로 표현 (예: `> 85%`, `== 0`, `< 100`)
- **알람 없는 지표**: "참고용" 또는 "-" 표기

---

## 카테고리별 주요 지표

### 메모리
- `redis_memory_used_bytes` — 실제 사용 메모리
- `redis_memory_max_bytes` — maxmemory 설정값
- `redis_mem_fragmentation_ratio` — 메모리 파편화 비율 (1.5 이상 주의)

### 복제 (Replication)
- `redis_connected_slaves` — 연결된 Replica 수
- `redis_replication_backlog_size` — 복제 백로그 크기
- `redis_master_repl_offset` — Master 복제 오프셋

### 클라이언트
- `redis_connected_clients` — 현재 연결 수
- `redis_blocked_clients` — BLPOP 등으로 블로킹된 클라이언트 수

### 성능
- `redis_commands_processed_total` — 총 처리 명령어 수
- `redis_keyspace_hits_total` — 캐시 히트 수
- `redis_keyspace_misses_total` — 캐시 미스 수
- `redis_slowlog_length` — Slow Log 항목 수

### 영속성
- `redis_rdb_last_bgsave_status` — 마지막 RDB 저장 성공 여부
- `redis_aof_enabled` — AOF 활성화 여부

### 클러스터
- `redis_cluster_enabled` — Cluster 모드 활성화
- `redis_cluster_state` — Cluster 상태 (ok=1)
- `redis_cluster_slots_assigned` — 할당된 슬롯 수 (정상: 16384)
