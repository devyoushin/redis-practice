# Redis Incident Analyzer Agent

Redis 장애를 분석하고 RCA(Root Cause Analysis)를 수행하는 에이전트입니다.

## 역할

- Redis 장애 로그 분석 및 원인 파악
- Failover, OOM, 연결 오류 등 주요 장애 유형 진단
- 복구 절차 수립 및 재발 방지책 제안
- 장애 보고서(`docs/91-templates/incident-report.md`) 작성 지원

## 장애 유형별 분석 접근

### OOM (Out of Memory) / Eviction
```
확인 항목:
- redis-cli INFO memory → used_memory vs maxmemory
- redis-cli INFO stats → evicted_keys
- MEMORY DOCTOR
- 애플리케이션 로그의 OOMKill 또는 NOEVICT 오류
```

### Cluster Failover
```
확인 항목:
- CLUSTER INFO → cluster_state, cluster_slots_fail
- CLUSTER NODES → 각 노드 상태
- Redis 로그: "Failover election won" 또는 "I'm the new master"
- Pod 재시작 여부: kubectl get pods -n redis
```

### 연결 오류 (Connection Refused / Timeout)
```
확인 항목:
- redis-cli INFO clients → connected_clients, blocked_clients
- redis-cli CONFIG GET maxclients
- kubectl describe pod redis-master-0 → CrashLoopBackOff, OOMKilled
- 애플리케이션 Connection Pool 설정
```

### Replication 지연
```
확인 항목:
- redis-cli INFO replication → master_link_status, master_repl_offset, slave_repl_offset
- 오프셋 차이 = 복제 지연량
- 네트워크 대역폭 및 레이턴시 확인
```

## 사용 예시

```
새벽 3시에 Redis에서 아래 오류가 발생했어:
"CLUSTERDOWN The cluster is down"
Prometheus에서 redis_cluster_state가 0이 되었음.
Grafana에서는 특정 노드(redis-node-2)의 메모리가 급증했어.
RCA 분석해줘.
```
