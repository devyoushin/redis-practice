# Redis Prometheus 지표

Redis는 `INFO` 명령어를 통해 수십 가지 지표를 제공합니다. Prometheus redis-exporter를 통해 수집하고, Grafana로 시각화합니다.

---

## 1. 개요

Redis 모니터링 핵심 영역:
- **메모리**: 사용량, 파편화, Eviction
- **성능**: 처리량, 지연, Hit Rate
- **복제**: 마스터-Replica 동기화 상태
- **연결**: 클라이언트 수, 거부 수
- **영속성**: RDB/AOF 저장 상태

---

## 2. 설명

### 2.1 핵심 개념

#### 카테고리별 핵심 지표

**메모리**

| Prometheus 지표 | INFO 항목 | 설명 | 알람 기준 |
|----------------|---------|------|---------|
| `redis_memory_used_bytes` | `used_memory` | 실제 사용 메모리 | maxmemory의 85% |
| `redis_memory_max_bytes` | `maxmemory` | 최대 메모리 설정값 | - |
| `redis_mem_fragmentation_ratio` | `mem_fragmentation_ratio` | 파편화 비율 | > 1.5 또는 < 1.0 |
| `redis_evicted_keys_total` | `evicted_keys` | Eviction된 키 수 | 지속 증가 |

**성능**

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_commands_processed_total` | 총 처리 명령어 수 | 급격한 감소 |
| `redis_keyspace_hits_total` | 캐시 히트 수 | Hit Rate < 80% |
| `redis_keyspace_misses_total` | 캐시 미스 수 | Hit Rate 계산용 |
| `redis_instantaneous_ops_per_sec` | 초당 처리 명령어 수 | 기준치 대비 이상 |
| `redis_slowlog_length` | Slow Log 항목 수 | > 0 지속 |

**복제**

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_connected_slaves` | 연결된 Replica 수 | < 예상 수 |
| `redis_master_repl_offset` | 마스터 복제 오프셋 | - |
| `redis_slave_repl_offset` | Replica 복제 오프셋 | 마스터와 차이 > 임계값 |

**연결**

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_connected_clients` | 현재 연결 수 | maxclients의 80% |
| `redis_blocked_clients` | 블로킹 클라이언트 수 | > 0 지속 |
| `redis_rejected_connections_total` | 거부된 연결 수 | > 0 |

**영속성**

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_rdb_last_bgsave_status` | RDB 저장 성공 여부 (ok=1) | != 1 |
| `redis_aof_enabled` | AOF 활성화 여부 | 운영환경 0이면 경고 |

### 2.2 실무 적용 코드

#### Prometheus Alert Rules

```yaml
groups:
  - name: redis-alerts
    rules:
      # 메모리 사용률 높음
      - alert: RedisMemoryHigh
        expr: |
          redis_memory_used_bytes / redis_config_maxmemory * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis 메모리 사용률 85% 초과"
          description: "Instance: {{ $labels.instance }}, 사용률: {{ $value | humanize }}%"

      # Eviction 발생
      - alert: RedisEviction
        expr: increase(redis_evicted_keys_total[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Redis Eviction 발생"
          description: "5분간 {{ $value }}개 키 Eviction 발생"

      # 연결 수 높음
      - alert: RedisConnectionsHigh
        expr: redis_connected_clients > 800
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis 연결 수 800 초과"

      # RDB 저장 실패
      - alert: RedisRdbSaveFailed
        expr: redis_rdb_last_bgsave_status != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis RDB 저장 실패"

      # Replica 연결 없음 (마스터에서)
      - alert: RedisReplicaDown
        expr: |
          redis_connected_slaves{role="master"} < 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis Replica 연결 없음"
```

#### Grafana 핵심 쿼리

```promql
# 메모리 사용률 (%)
redis_memory_used_bytes / redis_config_maxmemory * 100

# Hit Rate (%)
rate(redis_keyspace_hits_total[5m]) /
(rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) * 100

# 초당 명령어 처리 수
rate(redis_commands_processed_total[1m])

# 복제 지연 (오프셋 차이)
redis_master_repl_offset - redis_slave_repl_offset

# Eviction 속도 (초당)
rate(redis_evicted_keys_total[5m])

# 연결 사용률 (%)
redis_connected_clients / redis_config_maxclients * 100
```

#### redis-cli INFO로 직접 확인

```bash
# 전체 정보
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO all

# 메모리만
redis-cli INFO memory | grep -E "used_memory|maxmemory|fragmentation|evicted"

# 복제 상태
redis-cli INFO replication

# 성능 통계
redis-cli INFO stats | grep -E "hit|miss|evict|cmd"

# 클라이언트 연결
redis-cli INFO clients

# 영속성 상태
redis-cli INFO persistence | grep -E "rdb|aof"
```

### 2.3 Best Practice

- Hit Rate 80% 이하 지속 시 캐시 전략(TTL, 자료구조) 재검토
- 복제 오프셋 차이 모니터링으로 Replica 지연 조기 감지
- Eviction 알람은 절대값보다 **증가율** 기준으로 설정

---

## 3. 트러블슈팅

### 3.1 Hit Rate 급락

#### 증상
- `redis_keyspace_hits_total / (hits + misses)` 비율이 80% 아래로 하락

#### 원인
- TTL 짧게 설정 → 자주 만료
- maxmemory 부족으로 Eviction 과다
- 새 서비스 배포로 캐시 워밍업 필요

#### 해결 방법
```bash
# Eviction 확인
redis-cli INFO stats | grep evicted_keys

# TTL 분포 확인 (SCAN + TTL 샘플링)
redis-cli SCAN 0 COUNT 100 | xargs -I {} redis-cli TTL {}
```

### 3.2 초당 명령어 수 급감

#### 증상
- `rate(redis_commands_processed_total[1m])` 값이 정상 대비 50% 이하

#### 원인
- Redis 응답 지연 (Slow Log 증가)
- 연결 거부 (maxclients 초과)
- Redis Pod 재시작

#### 해결 방법
```bash
# Slow Log 확인
redis-cli SLOWLOG GET 10

# 연결 수 확인
redis-cli INFO clients | grep connected

# Pod 상태
kubectl get pods -n redis
```

---

## 4. 모니터링 및 확인

```bash
# 핵심 지표 원라이너 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> \
  INFO all | grep -E "used_memory_human|maxmemory_human|connected_clients|evicted_keys|keyspace_hits|keyspace_misses|rdb_last_bgsave_status|connected_slaves"
```

---

## 5. TIP

- `INFO everything` 대신 `INFO memory`, `INFO stats` 등 섹션별 조회로 부하 최소화
- Grafana 공식 Redis 대시보드 ID: `11835` (redis-exporter 기반)
- `redis-cli MONITOR`로 실시간 명령어 추적 가능 (운영 환경 주의 — 성능 영향)
- 참고: [Redis INFO Command](https://redis.io/commands/info/)
