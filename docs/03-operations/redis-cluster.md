# Redis Cluster

Redis Cluster(레디스 클러스터)는 데이터를 여러 노드에 자동 샤딩(Sharding)하여 수평 확장을 지원합니다. 총 16384개의 Hash Slot을 마스터 노드에 분산하며, 각 마스터는 하나 이상의 Replica를 가집니다.

---

## 1. 개요

Redis Cluster 핵심 특성:
- **샤딩**: 16384 Hash Slot을 N개 마스터에 균등 분배
- **고가용성**: 마스터 장애 시 Replica가 자동 승격
- **쿼럼**: 마스터 과반수 이상 장애 시 `CLUSTERDOWN` 상태
- **제약**: 다중 키 명령(`MGET`, `MSET`)은 동일 슬롯의 키만 허용

---

## 2. 설명

### 2.1 핵심 개념

#### Hash Slot 분배

```
마스터 3개 구성 예시:
  Master-0: Slot 0 ~ 5460
  Master-1: Slot 5461 ~ 10922
  Master-2: Slot 10923 ~ 16383

키 → 슬롯 계산: CRC16(key) % 16384
```

#### 클러스터 상태

| 상태 | 설명 | 조치 |
|------|------|------|
| `ok` | 정상 동작 | - |
| `fail` | 마스터 과반수 장애 | 마스터 복구 또는 클러스터 재구성 |

#### Hash Tag (같은 슬롯으로 강제 라우팅)

```redis-cli
# {} 안의 문자열로 슬롯 결정 — 같은 슬롯으로 강제
MSET {user:123}:name "홍길동" {user:123}:email "hong@test.com"
MGET {user:123}:name {user:123}:email
```

### 2.2 실무 적용 코드

#### Bitnami Redis Cluster Helm 배포

```yaml
# redis-cluster-values.yaml
cluster:
  enabled: true
  nodes: 6                          # 마스터 3 + Replica 3
  replicas: 1                       # 마스터당 Replica 수

auth:
  enabled: true
  password: ""                      # Kubernetes Secret 관리 권장

persistence:
  enabled: true
  storageClass: gp3
  size: 10Gi

redis:
  resources:
    requests:
      memory: 256Mi
      cpu: "100m"
    limits:
      memory: 512Mi
      cpu: "500m"
  config: |
    maxmemory 400mb
    maxmemory-policy allkeys-lru
    save ""                         # RDB 비활성화 (Cluster 모드 성능 최적화)
    appendonly yes
    appendfsync everysec
```

```bash
# Helm 배포
helm install redis bitnami/redis-cluster \
  -n redis \
  -f redis-cluster-values.yaml \
  --set auth.password=$(kubectl get secret redis -n redis -o jsonpath='{.data.redis-password}' | base64 -d)
```

#### 클러스터 상태 확인

```bash
# 클러스터 노드 상태
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER INFO

# 노드별 슬롯 분배 확인
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER NODES

# 슬롯별 키 수 확인
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER KEYSLOT mykey
```

#### 클러스터 Failover 수동 실행

```bash
# 특정 Replica를 마스터로 승격 (계획된 Failover)
kubectl exec -it redis-redis-cluster-1 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER FAILOVER

# 강제 Failover (마스터 응답 없을 때)
kubectl exec -it redis-redis-cluster-1 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER FAILOVER FORCE
```

#### 노드 추가 (스케일 아웃)

```bash
# 새 노드를 클러스터에 추가
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> \
  CLUSTER MEET <NEW_NODE_IP> 6379

# 슬롯 리샤딩 (기존 노드에서 새 노드로 슬롯 이동)
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli --cluster reshard \
  redis-redis-cluster-0.redis.svc.cluster.local:6379 \
  -a <PASSWORD>
```

#### 핫 키(Hot Key) 확인

```bash
# 핫 키 분석 (샘플링 모드)
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> --hotkeys

# 특정 노드의 명령어 통계
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> INFO commandstats | grep -v "calls=0"
```

### 2.3 Best Practice

- 마스터 수는 홀수로 설정 (3, 5, 7) — 쿼럼 확보
- Hash Tag `{}`는 꼭 필요한 경우만 사용 — 과도한 사용 시 핫 슬롯 발생
- Cluster 모드에서 `KEYS *` 사용 금지 → `SCAN` 사용
- 마스터당 Replica 최소 1개 이상 — 마스터 장애 시 자동 Failover

---

## 3. 트러블슈팅

### 3.1 CLUSTERDOWN 오류

#### 증상
```
CLUSTERDOWN The cluster is down
```

#### 원인
- 마스터 과반수(절반 초과) 장애 → 클러스터 쓰기 중단

#### 해결 방법
```bash
# 클러스터 상태 확인
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER INFO | grep cluster_state

# 장애 노드 확인
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER NODES | grep fail

# Pod 상태 확인
kubectl get pods -n redis -l app.kubernetes.io/name=redis-cluster

# 장애 Pod 재시작
kubectl delete pod redis-redis-cluster-2 -n redis
```

### 3.2 MOVED 오류 (슬롯 불일치)

#### 증상
```
MOVED 5678 redis-node-2:6379
```

#### 원인
- 클라이언트가 잘못된 노드로 요청 전송
- 클러스터 클라이언트가 아닌 단순 Redis 클라이언트 사용

#### 해결 방법
- Cluster 지원 클라이언트 사용 (Lettuce: `RedisClusterClient`, Jedis: `JedisCluster`)
- `redis-cli -c` 옵션으로 자동 MOVED 리다이렉션

```bash
# redis-cli Cluster 모드 (-c 옵션)
redis-cli -c -h redis-redis-cluster-0.redis.svc.cluster.local -p 6379 -a <PASSWORD>
```

### 3.3 슬롯 불균형 (특정 마스터 과부하)

#### 증상
- 특정 마스터 노드의 CPU/메모리만 높음
- 다른 노드는 여유 있음

#### 원인
- Hash Tag 과도 사용으로 특정 슬롯에 키 집중
- 초기 클러스터 구성 시 슬롯 분배 불균형

#### 해결 방법
```bash
# 슬롯별 키 분포 확인
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER SLOTS

# 슬롯 리샤딩으로 균등 분배
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli --cluster rebalance \
  redis-redis-cluster-0.redis.svc.cluster.local:6379 \
  -a <PASSWORD> --cluster-use-empty-masters
```

---

## 4. 모니터링 및 확인

```bash
# 클러스터 전체 상태
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER INFO

# 노드별 복제 상태
kubectl exec -it redis-redis-cluster-0 -n redis -- \
  redis-cli -a <PASSWORD> CLUSTER NODES

# 메모리 사용량 (전체 노드)
for i in 0 1 2 3 4 5; do
  echo "=== Node $i ===";
  kubectl exec -it redis-redis-cluster-$i -n redis -- \
    redis-cli -a <PASSWORD> INFO memory | grep used_memory_human;
done
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_cluster_state` | Cluster 상태 (ok=1) | != 1 |
| `redis_cluster_slots_assigned` | 할당된 슬롯 수 | != 16384 |
| `redis_cluster_slots_fail` | 장애 슬롯 수 | > 0 |
| `redis_connected_slaves` | 연결된 Replica 수 | 마스터당 < 1 |

---

## 5. TIP

- `redis-cli --cluster check <host>:<port>` 명령어로 클러스터 일관성 검사
- Bitnami Helm Chart에서 Replica 수 변경: `cluster.replicas` 값 수정 후 `helm upgrade`
- 클러스터 모드에서 Pub/Sub는 모든 노드에 브로드캐스트 — 대규모 환경에서 주의
- 참고: [Redis Cluster](https://redis.io/docs/management/scaling/)
