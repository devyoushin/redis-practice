# 네트워크 대역폭 병목과 스케일링 안티패턴

EKS Pod 증설 시 Redis 노드의 네트워크 대역폭(Network Bandwidth) 한계로 인해 지연이 발생하는 현상과, Pod/Node 수를 늘릴수록 커넥션 폭증으로 오히려 성능이 악화되는 스케일링 안티패턴을 다룸.

---

## 1. 개요

| 항목 | 설명 |
|------|------|
| 현상 | 특정 시간대 트래픽 급증 시 Redis 응답 지연 발생 |
| 초기 대응 | EKS Pod/Node 증설 (수평 스케일링) |
| 결과 | Redis 커넥션 폭증 → 네트워크 대역폭 초과 → 지연 악화 |
| 근본 원인 | EC2 인스턴스 타입별 네트워크 대역폭 상한 + 커넥션 수 비례 트래픽 증가 |

---

## 2. 설명

### 2.1 핵심 개념

#### 왜 Pod를 늘렸는데 Redis가 느려지는가?

```
[문제 상황 흐름]

1단계: 피크 트래픽 발생 → 애플리케이션 응답 지연
2단계: Pod 수평 확장 (HPA 또는 수동) → Pod 10개 → 30개
3단계: 각 Pod가 Redis에 Connection Pool 생성
       → 10개 × max-active 10 = 100 커넥션
       → 30개 × max-active 10 = 300 커넥션
4단계: Redis 커넥션 증가 → 요청 처리량 증가 → 네트워크 I/O 급증
5단계: EC2 인스턴스 네트워크 대역폭 상한 도달 → 패킷 지연/드롭
6단계: Redis 응답 지연 → 애플리케이션 타임아웃 → Pod 더 증설 → 악순환
```

#### EC2 인스턴스 타입별 네트워크 대역폭 한계

Redis가 실행되는 EC2 노드의 인스턴스 타입에 따라 네트워크 대역폭 상한이 정해져 있음. 이 한도를 초과하면 Redis 자체 성능과 무관하게 지연이 발생함.

| 인스턴스 타입 | 네트워크 대역폭 | 기준 대역폭 | 버스트 한도 |
|--------------|----------------|------------|-----------|
| `r6g.large` | Up to 10 Gbps | 0.75 Gbps | 10 Gbps |
| `r6g.xlarge` | Up to 10 Gbps | 1.25 Gbps | 10 Gbps |
| `r6g.2xlarge` | Up to 10 Gbps | 2.5 Gbps | 10 Gbps |
| `r6g.4xlarge` | Up to 10 Gbps | 5 Gbps | 10 Gbps |
| `r7g.xlarge` | Up to 12.5 Gbps | 1.25 Gbps | 12.5 Gbps |
| `r7g.2xlarge` | Up to 12.5 Gbps | 2.5 Gbps | 12.5 Gbps |

> **핵심**: "Up to 10 Gbps"는 버스트(Burst) 한도이며, 지속적으로 사용 가능한 기준 대역폭(Baseline Bandwidth)은 훨씬 낮음. 예를 들어 `r6g.large`의 기준 대역폭은 **0.75 Gbps (약 93 MB/s)**에 불과.

#### 네트워크 크레딧(Network Credit) 메커니즘

```
[EC2 네트워크 크레딧 동작 원리]

기준 대역폭 이하 사용 시:
  → 네트워크 크레딧 축적 (최대 버킷까지)

기준 대역폭 초과 사용 시 (버스트):
  → 축적된 크레딧 소모
  → 크레딧 소진 시 → 기준 대역폭으로 스로틀링(Throttling)

피크 트래픽이 지속되는 경우:
  → 초기 몇 분은 버스트로 처리
  → 크레딧 소진 후 기준 대역폭으로 제한
  → 이 시점부터 Redis 응답 지연 급격히 증가
```

#### 커넥션 증가가 네트워크 대역폭에 미치는 영향

```
[커넥션당 네트워크 오버헤드]

1 커넥션 = TCP 연결 유지 비용 + 요청/응답 패킷
  - TCP Keepalive: 주기적 패킷
  - Redis RESP 프로토콜 오버헤드
  - 매 명령어마다 왕복(RTT) 발생

커넥션 수 × 초당 명령어 수 × 평균 응답 크기 = 네트워크 처리량

예시:
  300 커넥션 × 100 cmd/s × 1KB 평균 응답 = ~30 MB/s (약 240 Mbps)
  → r6g.large 기준 대역폭(0.75 Gbps)의 32% 소비 (요청만 계산, 실제는 더 큼)

대용량 값 (Hash, List 등) 반환 시:
  300 커넥션 × 50 cmd/s × 10KB 평균 응답 = ~150 MB/s (약 1.2 Gbps)
  → r6g.large 기준 대역폭(0.75 Gbps) 즉시 초과!
```

#### 악순환 구조 (Cascade Failure)

```
                   ┌─────────────────────────────────────────┐
                   │                                         │
                   ▼                                         │
           트래픽 급증                                       │
               │                                             │
               ▼                                             │
       애플리케이션 응답 지연                                │
               │                                             │
               ▼                                             │
       HPA/수동으로 Pod 증설                                 │
               │                                             │
               ▼                                             │
       Redis 커넥션 폭증                                     │
       (Pod수 × max-active)                                  │
               │                                             │
               ▼                                             │
       Redis 노드 네트워크 대역폭 초과                      │
               │                                             │
               ▼                                             │
       Redis 응답 지연 더 악화 ──────────────────────────────┘
```

### 2.2 실무 적용 코드

#### 현재 네트워크 사용량 확인

```bash
# Redis 노드의 EC2 인스턴스 타입 확인
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
TYPE:.metadata.labels.node\\.kubernetes\\.io/instance-type,\
ZONE:.metadata.labels.topology\\.kubernetes\\.io/zone

# Redis Pod가 실행 중인 노드 확인
kubectl get pod redis-master-0 -n redis -o wide

# EC2 인스턴스의 네트워크 성능 확인 (AWS CLI)
aws ec2 describe-instance-types \
  --instance-types r6g.xlarge \
  --query 'InstanceTypes[0].NetworkInfo.{Bandwidth:NetworkCards[0].BaselineBandwidthInGbps,Peak:NetworkCards[0].PeakBandwidthInGbps}' \
  --output table
```

#### CloudWatch에서 네트워크 대역폭 확인

```bash
# EC2 인스턴스 네트워크 사용량 조회 (최근 1시간, 5분 단위)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkOut \
  --dimensions Name=InstanceId,Value=<INSTANCE_ID> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average Maximum \
  --unit Bytes

# NetworkBandwidthOutAllowanceExceeded 확인 (대역폭 초과 여부)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkBandwidthOutAllowanceExceeded \
  --dimensions Name=InstanceId,Value=<INSTANCE_ID> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum
```

> **핵심 지표**: `NetworkBandwidthOutAllowanceExceeded` 값이 0보다 크면 해당 시간 동안 네트워크 대역폭 초과로 인한 스로틀링이 발생한 것임.

#### Redis 커넥션 및 네트워크 통계 확인

```bash
# 현재 Redis 커넥션 수 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO clients
# connected_clients:300       ← Pod 증설 후 폭증 여부 확인
# maxclients:10000

# 네트워크 I/O 통계
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO stats | \
  grep -E "total_net|instantaneous"
# total_net_input_bytes:xxxxx
# total_net_output_bytes:xxxxx
# instantaneous_input_kbps:xxxx    ← 현재 입력 대역폭 (kbps)
# instantaneous_output_kbps:xxxx   ← 현재 출력 대역폭 (kbps)

# 초당 명령어 처리량
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO stats | \
  grep instantaneous_ops_per_sec
```

#### 커넥션 풀 사이즈 최적화

```yaml
# application.yml — Pod 증설 시에도 총 커넥션 수 제어
spring:
  data:
    redis:
      host: redis-master.redis.svc.cluster.local
      port: 6379
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          enabled: true
          max-active: 5              # 10 → 5로 축소 (Pod 수 증가 대비)
          max-idle: 3
          min-idle: 1
          max-wait: 1000ms
```

```
[총 커넥션 수 계산]

변경 전: Pod 30개 × max-active 10 = 300 커넥션
변경 후: Pod 30개 × max-active 5  = 150 커넥션

목표: 총 커넥션 수를 Redis maxclients의 60~70% 이내로 유지
```

#### 응답 크기 최적화로 네트워크 사용량 절감

```redis-cli
# 대용량 키 탐지
redis-cli --bigkeys

# 특정 키 크기 확인
MEMORY USAGE user:session:12345

# Hash에서 전체 대신 필요한 필드만 조회
# BAD: 전체 Hash 반환 (큰 네트워크 부하)
HGETALL user:profile:12345

# GOOD: 필요한 필드만 조회 (네트워크 절감)
HMGET user:profile:12345 name email
```

#### 인스턴스 타입 업그레이드 (근본 해결)

```yaml
# EKS Node Group 설정 변경 — Redis 전용 노드 그룹
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: ap-northeast-2
managedNodeGroups:
  - name: redis-nodes
    instanceType: r6g.2xlarge          # 기준 대역폭 2.5 Gbps
    desiredCapacity: 3
    labels:
      workload: redis
    taints:
      - key: dedicated
        value: redis
        effect: NoSchedule
```

```yaml
# Bitnami Redis Helm values — nodeSelector/tolerations 추가
master:
  nodeSelector:
    workload: redis
  tolerations:
    - key: dedicated
      operator: Equal
      value: redis
      effect: NoSchedule
  resources:
    requests:
      cpu: "2"
      memory: 8Gi
    limits:
      cpu: "4"
      memory: 8Gi

replica:
  nodeSelector:
    workload: redis
  tolerations:
    - key: dedicated
      operator: Equal
      value: redis
      effect: NoSchedule
```

### 2.3 Best Practice

- **총 커넥션 수 관리**: `Pod 수 × max-active` 합계가 Redis 노드의 네트워크 수용 범위 내인지 항상 확인
- **HPA maxReplicas 제한**: 무제한 스케일아웃 방지 — Redis 커넥션 한계를 역산하여 maxReplicas 설정
- **Redis 전용 노드 그룹**: 네트워크 대역폭이 충분한 인스턴스 타입으로 분리 운영
- **응답 크기 최소화**: `HGETALL` 대신 `HMGET`, 큰 List 대신 `LRANGE` 범위 제한
- **Lettuce 멀티플렉싱 활용**: Lettuce 기본 단일 연결 모드는 커넥션 수를 대폭 줄임 — Pool 활성화 시에만 커넥션 증가
- **Pipeline/Batch 사용**: 여러 명령을 묶어 전송하면 RTT 오버헤드와 네트워크 왕복 감소
- **Read Replica 활용**: 읽기 트래픽을 Replica로 분산하여 Master 노드 대역폭 부하 경감

---

## 3. 트러블슈팅

### 3.1 피크 시간대 Redis 응답 지연 급증

#### 증상
- 특정 시간대(점심, 저녁 등)에 Redis 응답 시간이 수십~수백 ms로 증가
- `SLOWLOG`에는 느린 명령어가 없음 — Redis 자체 처리는 빠르지만 응답이 느림
- CloudWatch에서 `NetworkBandwidthOutAllowanceExceeded > 0` 확인

#### 원인
- EC2 인스턴스의 네트워크 기준 대역폭 초과
- 버스트 크레딧 소진 후 기준 대역폭으로 스로틀링
- Redis 처리 시간은 마이크로초 단위이지만, 패킷 전송에서 밀리초 단위 지연 발생

#### 해결 방법
```bash
# 1. 대역폭 초과 여부 확인
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkBandwidthOutAllowanceExceeded \
  --dimensions Name=InstanceId,Value=<INSTANCE_ID> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Sum

# 2. Redis 네트워크 사용량 확인
redis-cli INFO stats | grep -E "instantaneous_output_kbps|instantaneous_input_kbps"

# 3. 즉시 조치: 인스턴스 타입 업그레이드
# r6g.large (기준 0.75 Gbps) → r6g.2xlarge (기준 2.5 Gbps) 이상으로 변경
```

### 3.2 Pod 증설 후 오히려 Redis 성능 악화

#### 증상
- 애플리케이션 지연 해결을 위해 Pod/Node 증설
- 증설 직후 Redis `connected_clients` 급증 (예: 100 → 300)
- Redis 응답 지연이 오히려 증가, 타임아웃 빈발

#### 원인
- Pod 증설 → 각 Pod가 `min-idle` 커넥션 즉시 생성 → 커넥션 수 비례 증가
- 커넥션 증가 → Redis 네트워크 I/O 증가 → 노드 네트워크 대역폭 초과
- 대역폭 초과 → 모든 커넥션의 응답 지연 → 커넥션 풀 고갈 → 타임아웃

#### 해결 방법
```bash
# 1. 현재 커넥션 수 확인
redis-cli INFO clients | grep connected_clients

# 2. 커넥션 풀 사이즈 축소 (애플리케이션 배포)
# max-active: 10 → 3~5로 축소
# min-idle: 2 → 1로 축소

# 3. 불필요한 유휴 커넥션 정리
redis-cli CLIENT LIST | awk '{print $1}' | head -20
# idle 시간이 긴 커넥션 식별 후 정리

# 4. HPA maxReplicas 역산 설정
# Redis maxclients: 10000
# 목표 커넥션 상한: 500 (maxclients의 5%)
# Pod당 max-active: 5
# → maxReplicas = 500 / 5 = 100
```

```yaml
# HPA에 maxReplicas 제한 추가
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 5
  maxReplicas: 50           # Redis 커넥션 한계 역산 기반 제한
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### 3.3 Lettuce 단일 연결에서 멀티플렉싱 병목

#### 증상
- Lettuce 기본 모드(단일 연결)에서 높은 동시성 요청 시 응답 지연
- 커넥션 수는 적지만 단일 TCP 연결의 처리량 한계에 도달

#### 원인
- Lettuce 단일 연결은 하나의 TCP 소켓으로 모든 요청을 멀티플렉싱
- 극단적 동시성(수천 req/s per Pod)에서는 단일 소켓의 I/O 버퍼가 병목

#### 해결 방법
```yaml
# Lettuce 풀 활성화 (적절한 크기로)
spring:
  data:
    redis:
      lettuce:
        pool:
          enabled: true
          max-active: 3          # 소규모 풀 (Pod 수 고려)
          max-idle: 2
          min-idle: 1
```

```
[Lettuce 풀 사이즈 가이드라인]

단일 연결 모드: 1 커넥션 per Pod → 총 커넥션 = Pod 수
풀 활성화: max-active 커넥션 per Pod → 총 커넥션 = Pod 수 × max-active

Pod 50개 기준:
  - 단일 연결: 50 커넥션 (네트워크 부하 최소)
  - max-active 3: 150 커넥션 (적절한 균형)
  - max-active 10: 500 커넥션 (대역폭 주의 필요)
```

---

## 4. 모니터링 및 확인

```bash
# 핵심 진단 명령어 모음
# 1) Redis 커넥션 수
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO clients | \
  grep -E "connected_clients|blocked_clients|maxclients"

# 2) Redis 네트워크 I/O (실시간)
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO stats | \
  grep -E "instantaneous_input_kbps|instantaneous_output_kbps|total_net"

# 3) EC2 네트워크 대역폭 초과 여부
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkBandwidthOutAllowanceExceeded \
  --dimensions Name=InstanceId,Value=<INSTANCE_ID> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Sum
```

| Prometheus / CloudWatch 지표 | 설명 | 알람 기준 |
|------------------------------|------|---------|
| `redis_connected_clients` | 현재 Redis 커넥션 수 | 예상치 대비 2배 이상 급증 시 |
| `redis_instantaneous_output_kbps` | Redis 출력 대역폭 (kbps) | 노드 기준 대역폭의 70% |
| `redis_instantaneous_input_kbps` | Redis 입력 대역폭 (kbps) | 노드 기준 대역폭의 70% |
| `NetworkBandwidthOutAllowanceExceeded` (CloudWatch) | EC2 네트워크 대역폭 초과 패킷 수 | > 0 |
| `NetworkBandwidthInAllowanceExceeded` (CloudWatch) | EC2 인바운드 대역폭 초과 패킷 수 | > 0 |
| `node_network_transmit_bytes_total` (Node Exporter) | 노드 네트워크 송신 바이트 | 기준 대역폭의 80% |

### Grafana 대시보드 쿼리 예시

```promql
# Redis 출력 대역폭 추이 (Mbps)
redis_instantaneous_output_kbps / 1024

# Pod 수 대비 커넥션 수 비율
redis_connected_clients / kube_deployment_status_replicas{deployment="my-app"}

# EC2 네트워크 사용률 (Node Exporter 기준)
rate(node_network_transmit_bytes_total{device="eth0"}[5m]) * 8 / 1e9
# → Gbps 단위, 인스턴스 기준 대역폭과 비교
```

---

## 5. TIP

- **인스턴스 타입 선택 시 네트워크 대역폭을 반드시 확인**: Redis는 CPU/메모리보다 네트워크가 병목이 되는 경우가 많음. "Up to X Gbps"는 버스트이므로 기준 대역폭(Baseline) 기준으로 판단해야 함
- **`NetworkBandwidthOutAllowanceExceeded`는 반드시 모니터링**: 이 CloudWatch 지표가 0이 아닌 순간 네트워크 스로틀링이 발생 중이며, Redis 지연의 원인일 가능성이 높음
- **Pod 스케일링 전에 항상 총 커넥션 수를 역산**: `Pod 수 × max-active ≤ 네트워크 수용 가능 커넥션`인지 확인
- **ENA Express 활용 검토**: 같은 AZ 내 인스턴스 간 최대 25 Gbps 대역폭 지원 (r6g.xlarge 이상, 추가 비용 없음)
- **ElastiCache 전환 고려**: 네트워크 대역폭 관리가 어려운 경우, AWS ElastiCache는 인스턴스 타입에 맞는 네트워크 대역폭을 보장하고 Enhanced I/O Multiplexing으로 처리량을 최대 72% 향상시킴
- 참고: [EC2 Network Bandwidth](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html), [Redis Latency Diagnosis](https://redis.io/docs/management/optimization/latency/)
