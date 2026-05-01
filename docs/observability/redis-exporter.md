# Redis Exporter 설정

Redis Exporter(레디스 익스포터)는 Redis `INFO` 명령어를 Prometheus 형식으로 변환하는 메트릭 수집기입니다. `oliver006/redis_exporter`가 표준으로 사용됩니다.

---

## 1. 개요

Redis Exporter 구성:
- **컨테이너**: `oliver006/redis_exporter` (사이드카 또는 독립 배포)
- **포트**: 9121 (기본)
- **메트릭 엔드포인트**: `/metrics`
- **수집 방식**: `INFO` + `CONFIG GET` + `DBSIZE` 등 Redis 명령어 호출

---

## 2. 설명

### 2.1 핵심 개념

#### 배포 방식 비교

| 방식 | 설명 | 권장 환경 |
|------|------|---------|
| 사이드카 | Redis Pod에 exporter 컨테이너 추가 | 단일 Redis 인스턴스 |
| 독립 Deployment | 별도 Pod로 여러 Redis 모니터링 | 다수 Redis 인스턴스 |
| Bitnami Helm 내장 | `metrics.enabled: true` 설정 | **EKS Bitnami 환경 권장** |

#### 주요 수집 지표

| 지표 그룹 | 예시 지표 |
|---------|---------|
| 메모리 | `redis_memory_used_bytes`, `redis_mem_fragmentation_ratio` |
| 성능 | `redis_commands_processed_total`, `redis_keyspace_hits_total` |
| 클라이언트 | `redis_connected_clients`, `redis_blocked_clients` |
| 복제 | `redis_connected_slaves`, `redis_master_repl_offset` |
| 영속성 | `redis_rdb_last_bgsave_status`, `redis_aof_enabled` |
| 클러스터 | `redis_cluster_state`, `redis_cluster_slots_assigned` |

### 2.2 실무 적용 코드

#### Bitnami Helm에서 Redis Exporter 활성화

```yaml
# redis-sentinel-values.yaml
metrics:
  enabled: true
  image:
    repository: oliver006/redis_exporter
    tag: latest

  serviceMonitor:
    enabled: true
    namespace: monitoring
    interval: 30s
    scrapeTimeout: 10s
    labels:
      release: prometheus

  resources:
    requests:
      memory: 32Mi
      cpu: "50m"
    limits:
      memory: 64Mi
      cpu: "100m"
```

#### 독립 Deployment로 배포

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
  namespace: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-exporter
  template:
    metadata:
      labels:
        app: redis-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9121"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: redis-exporter
          image: oliver006/redis_exporter:latest
          env:
            - name: REDIS_ADDR
              value: "redis://redis-master.redis.svc.cluster.local:6379"
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-auth
                  key: redis-password
          ports:
            - containerPort: 9121
              name: metrics
          resources:
            requests:
              memory: 32Mi
              cpu: "50m"
            limits:
              memory: 64Mi
              cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: redis-exporter
  namespace: redis
  labels:
    app: redis-exporter
spec:
  selector:
    app: redis-exporter
  ports:
    - port: 9121
      targetPort: 9121
      name: metrics
```

#### Prometheus ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-exporter
  namespace: redis
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: redis-exporter
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```

#### TLS Redis에 연결 시 Exporter 설정

```yaml
env:
  - name: REDIS_ADDR
    value: "rediss://redis-master.redis.svc.cluster.local:6380"   # rediss:// = TLS
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-auth
        key: redis-password
  - name: REDIS_EXPORTER_TLS_CA_CERT_FILE
    value: /tls/ca.crt
  - name: REDIS_EXPORTER_SKIP_TLS_VERIFICATION
    value: "false"
volumeMounts:
  - name: redis-tls
    mountPath: /tls
    readOnly: true
volumes:
  - name: redis-tls
    secret:
      secretName: redis-tls
```

#### Redis Cluster Exporter 설정

```yaml
env:
  - name: REDIS_ADDR
    value: "redis://redis-redis-cluster-0.redis.svc.cluster.local:6379"
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-cluster-auth
        key: redis-password
  - name: REDIS_EXPORTER_CLUSTER_MASTER_ONLY
    value: "false"   # 모든 노드 수집
```

#### 메트릭 엔드포인트 직접 확인

```bash
# Exporter Pod 포트 포워딩
kubectl port-forward -n redis svc/redis-exporter 9121:9121

# 메트릭 확인
curl http://localhost:9121/metrics | grep -E "redis_memory|redis_connected|redis_keyspace"

# 특정 Redis 인스턴스 메트릭 조회 (멀티 타겟 모드)
curl "http://localhost:9121/scrape?target=redis://redis-master.redis.svc.cluster.local:6379"
```

### 2.3 Best Practice

- Bitnami Helm 사용 시 `metrics.enabled: true` + `metrics.serviceMonitor.enabled: true`로 한 번에 설정
- 수집 간격은 30초 권장 — 초당 명령어가 많은 환경에서는 Redis에 부하 추가 주의
- 여러 Redis 인스턴스 모니터링 시 멀티 타겟 모드 활용 (단일 Exporter로 여러 Redis 수집)

---

## 3. 트러블슈팅

### 3.1 Exporter가 Redis에 연결 실패

#### 증상
- `/metrics` 엔드포인트에서 `redis_up 0` 반환

#### 원인
- Redis 주소/포트 오류
- 비밀번호 불일치
- 네트워크 접근 차단

#### 해결 방법
```bash
# Exporter 로그 확인
kubectl logs -n redis -l app=redis-exporter | tail -20

# Exporter Pod에서 Redis 연결 테스트
kubectl exec -it $(kubectl get pod -n redis -l app=redis-exporter -o jsonpath='{.items[0].metadata.name}') -n redis -- \
  redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> PING

# 메트릭 직접 확인
kubectl port-forward -n redis svc/redis-exporter 9121:9121
curl http://localhost:9121/metrics | grep redis_up
```

### 3.2 ServiceMonitor가 Prometheus에 인식되지 않음

#### 증상
- Prometheus Targets에 redis-exporter가 없음

#### 원인
- ServiceMonitor `labels`가 Prometheus `serviceMonitorSelector`와 불일치

#### 해결 방법
```bash
# Prometheus가 어떤 레이블로 ServiceMonitor를 선택하는지 확인
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'

# ServiceMonitor 레이블 수정
kubectl edit servicemonitor redis-exporter -n redis
# labels.release: prometheus  (Prometheus의 serviceMonitorSelector와 일치)
```

---

## 4. 모니터링 및 확인

```bash
# Exporter Pod 상태
kubectl get pods -n redis -l app=redis-exporter

# 메트릭 수집 확인
kubectl port-forward -n redis svc/redis-exporter 9121:9121
curl http://localhost:9121/metrics | grep redis_up

# Prometheus Target 확인
# Prometheus UI → Status → Targets → redis-exporter
```

| 확인 지표 | 설명 | 정상 값 |
|---------|------|--------|
| `redis_up` | Redis 연결 상태 | 1 |
| `redis_exporter_last_scrape_error` | 마지막 수집 오류 | 0 |
| `redis_exporter_scrapes_total` | 총 수집 횟수 | 지속 증가 |

---

## 5. TIP

- Grafana 공식 대시보드 ID: `11835` (redis-exporter 기반, Redis Overview)
- `REDIS_EXPORTER_LOG_FORMAT=json`으로 구조화 로그 출력 (로그 집계 용이)
- `REDIS_EXPORTER_CHECK_KEYS=mykey1,mykey2`로 특정 키 메모리 사용량 수집 가능
- 참고: [oliver006/redis_exporter](https://github.com/oliver006/redis_exporter)
