# 메모리 관리

Redis 메모리 관리는 `maxmemory` 설정과 `maxmemory-policy` Eviction(제거) 정책이 핵심입니다. 메모리 한도 초과 시 정책에 따라 키가 자동 삭제되며, 잘못된 정책은 서비스 장애로 이어집니다.

---

## 1. 개요

Redis 메모리 관리 핵심 설정:

| 설정 | 설명 | 운영 권장값 |
|------|------|-----------|
| `maxmemory` | Redis 최대 메모리 한도 | Pod 메모리의 70~80% |
| `maxmemory-policy` | 한도 초과 시 키 제거 정책 | `allkeys-lru` 또는 `volatile-lru` |
| `maxmemory-samples` | LRU/LFU 샘플 수 | 5~10 (높을수록 정확, 느림) |

---

## 2. 설명

### 2.1 핵심 개념

#### Eviction Policy(제거 정책) 비교

| 정책 | 설명 | 적합한 사용 사례 |
|------|------|--------------|
| `noeviction` | 메모리 꽉 차면 쓰기 오류 반환 | 데이터 유실 절대 불가 환경 |
| `allkeys-lru` | 전체 키 중 최근 미사용 순 제거 | 범용 캐시 **(권장)** |
| `volatile-lru` | TTL 있는 키 중 최근 미사용 순 제거 | 캐시 + 영구 데이터 혼용 |
| `allkeys-lfu` | 전체 키 중 사용 빈도 낮은 순 제거 | 사용 빈도 기반 캐시 (Redis 4.0+) |
| `volatile-lfu` | TTL 있는 키 중 사용 빈도 낮은 순 제거 | TTL + 빈도 기반 혼용 |
| `allkeys-random` | 전체 키 무작위 제거 | 균등한 랜덤 접근 패턴 |
| `volatile-random` | TTL 있는 키 무작위 제거 | - |
| `volatile-ttl` | TTL 짧은 키부터 제거 | 만료 임박 데이터 우선 제거 |

#### 메모리 파편화(Memory Fragmentation)

```
mem_fragmentation_ratio = used_memory_rss / used_memory

정상: 1.0 ~ 1.5
주의: > 1.5  → 메모리 파편화 과다 → MEMORY PURGE 고려
위험: < 1.0  → 스왑(Swap) 사용 중 → 즉시 조치 필요
```

#### 메모리 사용량 구성

```
used_memory           = Redis가 실제 사용하는 메모리
used_memory_rss       = OS가 Redis 프로세스에 할당한 메모리
used_memory_peak      = 역대 최고 메모리 사용량
mem_fragmentation_ratio = used_memory_rss / used_memory
```

### 2.2 실무 적용 코드

#### 메모리 설정 (redis.conf)

```
# 최대 메모리 설정
maxmemory 4gb                    # Pod 메모리 6GB 기준 약 67%

# Eviction 정책
maxmemory-policy allkeys-lru     # 전체 키 LRU (범용 캐시 권장)

# LRU 샘플 수 (정확도 vs 성능 트레이드오프)
maxmemory-samples 5              # 기본값 5 (일반적으로 충분)

# 메모리 보고 단위
active-expire-enabled yes        # 만료 키 즉시 정리 활성화
lazyfree-lazy-eviction yes       # 비동기 Eviction (응답 지연 최소화)
lazyfree-lazy-expire yes         # 비동기 만료 처리
```

#### 메모리 상태 진단

```bash
# 전체 메모리 정보
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO memory

# 메모리 진단 (문제 자동 감지)
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> MEMORY DOCTOR

# 특정 키 메모리 사용량
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> MEMORY USAGE product:cache:12345

# 메모리 사용량 Top N 키 (Redis 4.0+)
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> MEMORY USAGE --samples 0

# 전체 통계 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO stats | grep -E "evicted|expired"
```

#### Eviction 실시간 확인

```bash
# Eviction 발생 여부 확인
redis-cli INFO stats | grep evicted_keys

# 메모리 사용률 계산
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio"
```

#### 메모리 파편화 해소

```bash
# 메모리 파편화 정리 (Redis 4.0+, 비동기)
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> MEMORY PURGE

# 또는 자동 파편화 정리 활성화
redis-cli CONFIG SET activedefrag yes              # 자동 파편화 정리
redis-cli CONFIG SET active-defrag-threshold-lower 10  # 파편화 10% 이상 시 시작
redis-cli CONFIG SET active-defrag-threshold-upper 25  # 25% 이상 시 최대 속도
```

#### 대용량 키 탐지

```bash
# 대용량 키 탐지 (redis-cli --bigkeys)
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> --bigkeys

# SCAN + MEMORY USAGE로 특정 패턴 키 메모리 분석
redis-cli SCAN 0 MATCH "product:*" COUNT 100 | while read keys; do
  for key in $keys; do
    echo "$key: $(redis-cli MEMORY USAGE $key)"
  done
done
```

### 2.3 Best Practice

- `maxmemory`는 Pod 메모리의 70~80%로 설정 — OOM Killer 방지 여유 확보
- 캐시 전용이면 `allkeys-lru`, 캐시+영구 데이터 혼용이면 `volatile-lru`
- `lazyfree-lazy-eviction yes`로 Eviction 시 응답 지연 최소화
- 파편화 비율 1.5 초과 시 `MEMORY PURGE` 또는 Redis 재시작 계획

---

## 3. 트러블슈팅

### 3.1 OOM Killer가 Redis 프로세스 종료

#### 증상
- Redis Pod가 갑자기 재시작 (`OOMKilled` 상태)
- `kubectl describe pod redis-master-0`에서 `OOMKilled` 확인

#### 원인
- `maxmemory`를 Pod 메모리 한도에 너무 가깝게 설정
- AOF Buffer, Copy-on-Write 오버헤드로 실제 사용량이 `maxmemory` 초과

#### 해결 방법
```bash
# Pod 메모리 한도 확인
kubectl describe pod redis-master-0 -n redis | grep -A 5 "Limits"

# maxmemory를 Pod 메모리의 70%로 조정
# Pod limits: 512Mi → maxmemory: 350mb
redis-cli CONFIG SET maxmemory 350mb

# Pod limits 자체 증가 (Bitnami values.yaml)
# master.resources.limits.memory: 1Gi
```

### 3.2 Eviction 과다 발생 (캐시 효율 저하)

#### 증상
- `evicted_keys` 값이 지속 증가
- 캐시 Hit Rate 저하 (DB 부하 증가)

#### 원인
- `maxmemory` 한도가 너무 낮음
- 불필요한 대용량 키 또는 TTL 없는 키 존재

#### 해결 방법
```bash
# Eviction 확인
redis-cli INFO stats | grep evicted_keys

# maxmemory 증가 (즉시 적용)
redis-cli CONFIG SET maxmemory 6gb

# 불필요한 키 정리
redis-cli --scan --pattern "old:cache:*" | xargs redis-cli DEL

# 대용량 키 탐지 및 최적화
redis-cli --bigkeys
```

### 3.3 메모리 파편화 과다 (fragmentation_ratio > 2.0)

#### 증상
- `mem_fragmentation_ratio` 값이 2.0 이상
- 실제 데이터는 적은데 메모리 사용량 높음

#### 원인
- 잦은 키 생성/삭제로 메모리 파편화
- 다양한 크기의 키 혼재

#### 해결 방법
```bash
# 파편화 비율 확인
redis-cli INFO memory | grep mem_fragmentation_ratio

# 방법 1: MEMORY PURGE (즉시 정리, 일시적 블로킹 가능)
redis-cli MEMORY PURGE

# 방법 2: Active Defragmentation 활성화 (점진적 정리)
redis-cli CONFIG SET activedefrag yes

# 방법 3: Redis 재시작 (파편화 완전 해소)
kubectl delete pod redis-master-0 -n redis
```

---

## 4. 모니터링 및 확인

```bash
# 핵심 메모리 지표 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO memory | \
  grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio|evicted"
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_memory_used_bytes` | 사용 중 메모리 | maxmemory의 85% |
| `redis_mem_fragmentation_ratio` | 파편화 비율 | > 1.5 또는 < 1.0 |
| `redis_evicted_keys_total` | 누적 Eviction 키 수 | 지속 증가 시 |
| `redis_expired_keys_total` | 만료된 키 수 | 참고용 |

---

## 5. TIP

- `maxmemory 0`은 한도 없음 — 운영 환경 절대 금지
- LFU 정책(`allkeys-lfu`)은 Redis 4.0+에서만 지원 — 접근 빈도 기반 캐시에 더 적합
- `MEMORY DOCTOR` 명령어가 파편화/스왑/설정 문제를 자동 진단하여 조언 제공
- 참고: [Redis Memory Optimization](https://redis.io/docs/management/optimization/memory-optimization/)
