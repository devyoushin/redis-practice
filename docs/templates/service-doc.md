# {서비스명} Redis 구성 문서

{서비스명}에서 사용하는 Redis 구성 및 운영 가이드입니다.

---

## 1. 개요

### 서비스 정보

| 항목 | 값 |
|------|-----|
| 서비스명 | {서비스명} |
| Redis 용도 | {캐시 / 세션 / 메시지 큐 / 기타} |
| Redis 모드 | {Cluster / Sentinel / Standalone} |
| 접속 엔드포인트 | `redis-master.redis.svc.cluster.local:6379` |
| 사용 자료구조 | {String / Hash / List / Set / Sorted Set} |
| TTL 정책 | {기간 또는 없음} |

---

## 2. 설명

### 2.1 데이터 모델

#### Key 네이밍 규칙

```
{서비스}:{도메인}:{식별자}
예시:
  payment:session:{userId}       # 결제 세션
  product:cache:{productId}      # 상품 캐시
  rate:limit:{userId}:{date}     # Rate Limit 카운터
```

#### 자료구조 사용 목적

| Key 패턴 | 자료구조 | TTL | 용도 |
|---------|---------|-----|------|
| `{prefix}:{id}` | String | {시간} | {설명} |

### 2.2 Redis 클라이언트 설정

```yaml
# Spring Boot application.yml 예시
spring:
  data:
    redis:
      host: redis-master.redis.svc.cluster.local
      port: 6379
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          max-active: 10
          max-idle: 5
          min-idle: 2
          max-wait: 1000ms
        timeout: 2000ms
```

### 2.3 Best Practice

- TTL 없는 키 생성 금지 — 반드시 `EXPIRE` 설정
- 대용량 키(`value > 10KB`) 사용 시 사전 검토
- `KEYS *` 명령어 운영 환경 사용 금지 → `SCAN` 사용

---

## 3. 트러블슈팅

### 3.1 {증상}

#### 증상

```
{오류 메시지 또는 증상 설명}
```

#### 원인

- {원인}

#### 해결 방법

```redis-cli
# 진단 명령어
{명령어}
```

---

## 4. 모니터링 및 확인

```bash
# 서비스 키 수 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> \
  INFO keyspace

# 서비스 키 패턴 카운트
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> \
  SCAN 0 MATCH "{prefix}:*" COUNT 100
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_memory_used_bytes` | 메모리 사용량 | maxmemory의 85% |
| `redis_keyspace_hits_total` | 캐시 히트 | Hit Rate < 80% |

---

## 5. TIP

- {실운영 팁}
