# Redis Sentinel

Redis Sentinel(레디스 센티넬)은 Redis 고가용성(High Availability) 솔루션입니다. 마스터 장애를 감지하고 자동으로 Replica를 마스터로 승격(Failover)합니다. 샤딩이 필요 없고 단일 마스터로 충분한 환경에 적합합니다.

---

## 1. 개요

Redis Sentinel 구성:
- **마스터 1개** + **Replica N개**: 실제 데이터 저장
- **Sentinel 3개 이상**: 마스터 모니터링 및 Failover 결정
- **쿼럼(Quorum)**: Sentinel 과반수 동의 시 Failover 실행

---

## 2. 설명

### 2.1 핵심 개념

#### Sentinel vs Cluster 선택 기준

| 비교 항목 | Sentinel | Cluster |
|---------|---------|---------|
| 샤딩 | 없음 (단일 마스터) | 있음 (N개 마스터) |
| 수평 확장 | 불가 | 가능 |
| 최대 메모리 | 단일 서버 메모리 | 마스터 수 × 서버 메모리 |
| 클라이언트 복잡도 | 낮음 | 높음 (MOVED 처리) |
| 권장 환경 | 소/중규모, 세션/캐시 | 대규모, 고처리량 |

#### Failover 흐름

```
1. Sentinel이 마스터 응답 없음 감지 (down-after-milliseconds)
2. Sentinel들이 SDOWN(Subjective Down) 판정
3. 쿼럼 이상 Sentinel이 동의 → ODOWN(Objective Down) 선언
4. Leader Sentinel이 Failover 시작
5. Replica 중 가장 최신 데이터 보유 Replica 선택
6. 선택된 Replica: SLAVEOF NO ONE → 마스터 승격
7. 나머지 Replica들: 새 마스터 추종
8. 클라이언트: Sentinel에서 새 마스터 주소 조회 후 재연결
```

#### Sentinel 핵심 설정

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `down-after-milliseconds` | 30000 | 마스터 다운 판정 시간 (ms) |
| `failover-timeout` | 180000 | Failover 타임아웃 (ms) |
| `parallel-syncs` | 1 | Failover 후 동시 재동기화 Replica 수 |
| `quorum` | 2 | Failover 결정 최소 Sentinel 수 |

### 2.2 실무 적용 코드

#### Bitnami Redis Sentinel Helm 배포

```yaml
# redis-sentinel-values.yaml
architecture: replication             # Sentinel 모드

sentinel:
  enabled: true
  masterSet: mymaster                 # Sentinel 마스터 이름
  quorum: 2                           # Sentinel 3개 시 2 (과반수)
  downAfterMilliseconds: 10000        # 10초 내 응답 없으면 다운 판정
  failoverTimeout: 180000             # 3분 Failover 타임아웃
  parallelSyncs: 1

auth:
  enabled: true
  password: ""                        # Kubernetes Secret 관리 권장

replica:
  replicaCount: 2                     # 마스터 1 + Replica 2

master:
  persistence:
    enabled: true
    storageClass: gp3
    size: 10Gi
  resources:
    requests:
      memory: 256Mi
      cpu: "100m"
    limits:
      memory: 512Mi
      cpu: "500m"

replica:
  resources:
    requests:
      memory: 256Mi
      cpu: "100m"
    limits:
      memory: 512Mi
      cpu: "500m"
```

```bash
# Helm 배포
helm install redis bitnami/redis \
  -n redis \
  -f redis-sentinel-values.yaml
```

#### Sentinel 상태 확인

```bash
# Sentinel 정보 조회
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL masters

# 현재 마스터 주소 조회
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# Sentinel 쿼럼 상태
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL ckquorum mymaster

# Replica 상태
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL replicas mymaster
```

#### 수동 Failover 실행

```bash
# 계획된 마스터 전환 (마스터에서 실행)
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL failover mymaster

# Failover 후 새 마스터 확인
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

#### 복제 상태 확인

```bash
# 마스터에서 복제 상태 확인
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> INFO replication

# 출력 예시:
# role:master
# connected_slaves:2
# slave0:ip=10.0.0.2,port=6379,state=online,offset=1234567,lag=0
# slave1:ip=10.0.0.3,port=6379,state=online,offset=1234567,lag=1
```

#### Spring Boot Sentinel 클라이언트 설정

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes:
          - redis-node-0.redis.svc.cluster.local:26379
          - redis-node-1.redis.svc.cluster.local:26379
          - redis-node-2.redis.svc.cluster.local:26379
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          max-active: 10
          max-idle: 5
          min-idle: 2
```

### 2.3 Best Practice

- Sentinel 3개 이상 필수 (쿼럼 확보) — 2개 이하 시 Failover 불가
- `down-after-milliseconds`는 10000~30000ms 권장 — 너무 짧으면 네트워크 순간 지연으로 오탐
- 클라이언트는 Sentinel 주소를 직접 사용 — 마스터 IP 하드코딩 금지
- Replica를 읽기 전용으로 활용 시 `replica-serve-stale-data yes` 설정 확인

---

## 3. 트러블슈팅

### 3.1 Failover 발생 후 클라이언트 연결 오류

#### 증상
- Failover 완료 후에도 클라이언트가 이전 마스터 IP로 연결 시도
- `Connection refused` 오류

#### 원인
- 클라이언트가 Sentinel을 통한 마스터 주소 갱신을 하지 않음
- 마스터 IP 하드코딩

#### 해결 방법
- Sentinel 지원 클라이언트 사용 (Lettuce `SentinelConfiguration`, Jedis `JedisSentinelPool`)
- 애플리케이션 재시작 (임시 조치)

```bash
# 새 마스터 주소 확인
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

### 3.2 Sentinel 쿼럼 불충족

#### 증상
```
NOGOODSLAVE No suitable slave to promote
```
또는 Failover가 실행되지 않음

#### 원인
- Sentinel 수가 쿼럼(quorum) 미만
- Sentinel Pod 장애

#### 해결 방법
```bash
# Sentinel 상태 확인
kubectl get pods -n redis | grep redis-node

# 쿼럼 상태 확인
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL ckquorum mymaster

# 장애 Sentinel Pod 재시작
kubectl delete pod redis-node-2 -n redis
```

### 3.3 Replica 복제 지연

#### 증상
- `INFO replication`에서 `lag` 값이 크게 증가
- Failover 후 Replica가 오래된 데이터 보유

#### 원인
- 네트워크 대역폭 부족 또는 마스터 처리량 과다
- 복제 백로그(repl-backlog-size) 초과로 전체 재동기화 발생

#### 해결 방법
```bash
# 복제 지연 확인
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> INFO replication | grep -E "lag|offset"

# 복제 백로그 크기 증가
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> CONFIG SET repl-backlog-size 100mb
```

---

## 4. 모니터링 및 확인

```bash
# 복제 상태 실시간 모니터링
watch -n 5 "kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> INFO replication"

# Sentinel 마스터 정보
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL masters
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_connected_slaves` | 연결된 Replica 수 | < 예상 수 |
| `redis_replication_offset` | 복제 오프셋 차이 | 마스터-Replica 간 차이 > 임계값 |
| `redis_master_link_status` | 마스터 연결 상태 | != 1 (Replica에서) |

---

## 5. TIP

- Bitnami Sentinel 모드에서 마스터 접속: `redis-master.redis.svc.cluster.local:6379`
- Sentinel 포트: 26379 (Redis 기본 6379와 다름)
- `SENTINEL RESET <master-name>`으로 Sentinel 상태 초기화 (긴급 복구 시)
- 참고: [Redis Sentinel](https://redis.io/docs/management/sentinel/)
