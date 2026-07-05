# 연결 풀 최적화

Redis 연결 풀(Connection Pool)은 클라이언트와 Redis 서버 간 연결을 재사용하여 연결 생성 오버헤드를 줄입니다. Lettuce와 Jedis가 주요 Java 클라이언트이며, 각각 연결 모델이 다릅니다.

---

## 1. 개요

| 클라이언트 | 연결 모델 | 스레드 안전 | 권장 환경 |
|---------|---------|----------|---------|
| Lettuce | 비동기, 싱글 연결 공유 | 가능 | Spring Boot 기본, 일반 환경 |
| Jedis | 동기, 풀 기반 | 풀 사용 시 가능 | 동기 처리, 레거시 |

---

## 2. 설명

### 2.1 핵심 개념

#### Lettuce vs Jedis 연결 모델

```
Lettuce:
  단일 연결 → 멀티플렉싱(Multiplexing)으로 다중 요청 처리
  → 연결 수 자체가 적음
  → 풀 설정은 물리 연결 수 제한 (기본 불필요)

Jedis:
  요청마다 풀에서 연결 획득 → 처리 후 반납
  → 풀 크기 = 동시 처리 가능 요청 수
  → 풀 고갈 시 대기 또는 오류
```

#### 연결 수 계산 기준

```
Jedis 풀 최적 크기 = 동시 요청 수 × 평균 Redis 처리 시간 / 허용 지연

예시:
  동시 요청: 100 req/s
  Redis 처리 시간: 1ms
  → 최소 연결 수: 100 × 0.001 = 0.1 (여유: 10~20개)
```

#### 주요 풀 설정 항목

| 설정 | 설명 | 권장값 |
|------|------|--------|
| `max-active` | 최대 활성 연결 수 | CPU 코어 수 × 2 ~ 10 |
| `max-idle` | 최대 유휴 연결 수 | max-active의 50% |
| `min-idle` | 최소 유휴 연결 수 (미리 연결) | 2~5 |
| `max-wait` | 연결 획득 최대 대기 시간 | 1000~2000ms |
| `timeout` | 명령어 실행 타임아웃 | 2000ms |

### 2.2 실무 적용 코드

#### Spring Boot + Lettuce 설정

```yaml
# application.yml
spring:
  data:
    redis:
      host: redis-master.redis.svc.cluster.local
      port: 6379
      password: ${REDIS_PASSWORD}
      timeout: 2000ms                  # 명령어 타임아웃
      connect-timeout: 5000ms          # 연결 타임아웃
      lettuce:
        pool:
          enabled: true                # 풀 활성화 (기본 false — 단일 연결)
          max-active: 10               # 최대 연결 수
          max-idle: 5                  # 최대 유휴 연결
          min-idle: 2                  # 최소 유휴 연결 (미리 생성)
          max-wait: 1000ms             # 연결 대기 타임아웃
        shutdown-timeout: 100ms
```

```java
// Lettuce 고급 설정 (Java Config)
@Bean
public LettuceConnectionFactory redisConnectionFactory() {
    RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
    config.setHostName("redis-master.redis.svc.cluster.local");
    config.setPort(6379);
    config.setPassword(RedisPassword.of(redisPassword));

    GenericObjectPoolConfig<StatefulRedisConnection<String, String>> poolConfig =
        new GenericObjectPoolConfig<>();
    poolConfig.setMaxTotal(10);
    poolConfig.setMaxIdle(5);
    poolConfig.setMinIdle(2);
    poolConfig.setMaxWaitMillis(1000);

    LettucePoolingClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
        .poolConfig(poolConfig)
        .commandTimeout(Duration.ofMillis(2000))
        .shutdownTimeout(Duration.ofMillis(100))
        .build();

    return new LettuceConnectionFactory(config, clientConfig);
}
```

#### Spring Boot + Jedis 설정

```yaml
# application.yml — Jedis
spring:
  data:
    redis:
      host: redis-master.redis.svc.cluster.local
      port: 6379
      password: ${REDIS_PASSWORD}
      jedis:
        pool:
          max-active: 10
          max-idle: 5
          min-idle: 2
          max-wait: 1000ms
```

```xml
<!-- pom.xml — Jedis 사용 시 Lettuce 제외 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
  <exclusions>
    <exclusion>
      <groupId>io.lettuce</groupId>
      <artifactId>lettuce-core</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>redis.clients</groupId>
  <artifactId>jedis</artifactId>
</dependency>
```

#### Sentinel 모드 연결 설정

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

#### 연결 상태 확인

```bash
# 현재 연결 수 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO clients

# 출력:
# connected_clients:15          # 현재 연결 수
# blocked_clients:0             # 블로킹된 클라이언트 수
# maxclients:10000              # 최대 연결 한도

# 클라이언트 목록 (연결 당 상세)
redis-cli CLIENT LIST

# 최대 연결 수 조정
redis-cli CONFIG SET maxclients 1000
```

### 2.3 Best Practice

- Lettuce 기본 단일 연결은 대부분의 경우 충분 — 풀 필요 시 `pool.enabled: true`
- `min-idle`로 최소 연결을 미리 준비 → 콜드 스타트 지연 방지
- 애플리케이션 인스턴스 수 × max-active < Redis `maxclients` 확인
- 연결 타임아웃은 Redis 응답 시간 P99의 3~5배로 설정

---

## 3. 트러블슈팅

### 3.1 Connection Pool Exhausted

#### 증상
```
Unable to acquire connection from pool within the maximum wait time
io.lettuce.core.pool.ConnectionPoolExhaustedException
```

#### 원인
- `max-active` 한도 도달 → 새 연결 획득 대기 중 `max-wait` 초과
- Redis 처리 지연으로 연결이 반납되지 않음

#### 해결 방법
```yaml
# max-active 증가
lettuce:
  pool:
    max-active: 20        # 10 → 20으로 증가
    max-wait: 2000ms      # 대기 시간 증가
```

```bash
# Redis 응답 지연 원인 확인
redis-cli SLOWLOG GET 10
redis-cli INFO stats | grep -E "latency|cmd"
```

### 3.2 Connection Timeout

#### 증상
```
io.lettuce.core.RedisCommandTimeoutException: Command timed out after 2 second(s)
```

#### 원인
- Redis 처리 시간 > `timeout` 설정
- 네트워크 지연 또는 Redis 과부하

#### 해결 방법
```yaml
# 타임아웃 증가 (임시 조치)
spring:
  data:
    redis:
      timeout: 5000ms
```

```bash
# 현재 Redis 응답 지연 측정
redis-cli --latency -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD>

# Slow Log 확인
redis-cli SLOWLOG GET 20
```

### 3.3 연결 누수 (Connection Leak)

#### 증상
- `connected_clients` 가 시간이 지남에 따라 지속 증가
- 애플리케이션 재시작 없이 연결 수 감소하지 않음

#### 원인
- 예외 발생 시 연결 반납 누락 (try-with-resources 미사용)
- JedisPool에서 연결을 `close()` 하지 않음

#### 해결 방법
```bash
# 오래된 유휴 연결 강제 종료
redis-cli CLIENT KILL ID <client-id>

# 전체 클라이언트 연결 정보 확인
redis-cli CLIENT LIST
```

```java
// Jedis try-with-resources 패턴 (연결 자동 반납)
try (Jedis jedis = jedisPool.getResource()) {
    jedis.set("key", "value");
}  // 자동으로 풀에 반납
```

---

## 4. 모니터링 및 확인

```bash
# 연결 상태 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO clients

# 연결 지연 측정
redis-cli --latency -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD>

# 연결 통계 (초당 연결 생성/종료 수)
redis-cli INFO stats | grep -E "total_connections|rejected_connections"
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_connected_clients` | 현재 연결 수 | maxclients의 80% |
| `redis_blocked_clients` | 블로킹 클라이언트 수 | > 0 지속 시 |
| `redis_rejected_connections_total` | 거부된 연결 수 | > 0 |

---

## 5. TIP

- Lettuce는 기본적으로 풀 없이 싱글 연결 + 멀티플렉싱 사용 → 연결 수 자체가 적음
- `redis-cli --latency-history`로 연결 지연 추이 모니터링
- keepalive 설정으로 유휴 연결 유지: `tcp-keepalive 300` (redis.conf)
- 참고: [Lettuce Documentation](https://lettuce.io/), [Jedis Pool Config](https://github.com/redis/jedis)
