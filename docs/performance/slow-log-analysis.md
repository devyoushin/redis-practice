# Slow Log 분석

Redis Slow Log(슬로우 로그)는 설정한 임계값보다 오래 걸린 명령어를 기록합니다. O(N) 명령어, 대형 키 조회, 메모리 압박 상황을 진단하는 핵심 도구입니다.

---

## 1. 개요

Slow Log 핵심 설정:

| 설정 | 기본값 | 권장값 | 설명 |
|------|--------|--------|------|
| `slowlog-log-slower-than` | 10000 (10ms) | 1000 (1ms) | Slow Log 기록 임계값 (마이크로초) |
| `slowlog-max-len` | 128 | 256 | 최대 저장 항목 수 |

---

## 2. 설명

### 2.1 핵심 개념

#### Slow Log 항목 구성

```
SLOWLOG GET 결과 항목:
  1) ID (순번)
  2) Unix timestamp (발생 시각)
  3) 실행 시간 (마이크로초)
  4) 명령어 + 인자
  5) 클라이언트 IP:PORT
  6) 클라이언트 이름
```

#### O(N) 위험 명령어

| 명령어 | 복잡도 | 대안 |
|--------|--------|------|
| `KEYS pattern` | O(N) | `SCAN` |
| `SMEMBERS key` | O(N) | `SSCAN` |
| `HGETALL key` | O(N) | `HMGET` (필요한 필드만) |
| `LRANGE key 0 -1` | O(N) | 페이징 (`LRANGE 0 49`) |
| `SORT key` | O(N+M log M) | 사전 정렬 Sorted Set |
| `SUNION k1 k2` | O(N) | `SUNIONSTORE` 후 읽기 |
| `FLUSHDB` | O(N) | 운영 환경 금지 |

#### 지연 원인 분류

| 원인 | 증상 | 진단 방법 |
|------|------|---------|
| O(N) 명령어 | Slow Log에 KEYS/SMEMBERS 등장 | `SLOWLOG GET` |
| 대형 키(Big Key) | 특정 키 조회 시 지연 | `--bigkeys`, `MEMORY USAGE` |
| 메모리 파편화 | 전반적 지연 | `INFO memory` |
| AOF fsync 지연 | 쓰기 명령 지연 | `INFO persistence` |
| 네트워크 지연 | 클라이언트 측 지연 | `--latency` |

### 2.2 실무 적용 코드

#### Slow Log 설정 및 조회

```bash
# Slow Log 설정 확인
redis-cli CONFIG GET slowlog-log-slower-than
redis-cli CONFIG GET slowlog-max-len

# 임계값 변경 (1ms = 1000 마이크로초)
redis-cli CONFIG SET slowlog-log-slower-than 1000
redis-cli CONFIG SET slowlog-max-len 256

# Slow Log 조회 (최근 10개)
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> SLOWLOG GET 10

# Slow Log 항목 수 확인
redis-cli SLOWLOG LEN

# Slow Log 초기화
redis-cli SLOWLOG RESET
```

#### Slow Log 분석 예시

```
1) 1) (integer) 42                    # ID
   2) (integer) 1746014400            # Unix timestamp
   3) (integer) 15000                 # 실행 시간: 15ms (15000 마이크로초)
   4) 1) "KEYS"
      2) "*"                          # 위험! KEYS * 사용
   5) "10.0.0.5:54321"
   6) "my-service"

→ 조치: KEYS * → SCAN 으로 교체
```

#### SCAN으로 안전하게 키 순회

```bash
# KEYS * 대신 SCAN 사용 (블로킹 없음)
redis-cli SCAN 0 MATCH "product:cache:*" COUNT 100

# 전체 키 순회 스크립트 (배시)
cursor=0
while true; do
  result=$(redis-cli SCAN $cursor MATCH "product:*" COUNT 100)
  cursor=$(echo "$result" | head -1)
  keys=$(echo "$result" | tail -n +2)
  echo "$keys"
  if [ "$cursor" = "0" ]; then break; fi
done
```

#### 대형 키(Big Key) 탐지

```bash
# redis-cli --bigkeys로 타입별 최대 키 탐지
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> \
  --bigkeys

# 출력 예시:
# Biggest string found 'product:cache:large' has 102400 bytes
# Biggest hash   found 'user:data:massive' has 50000 fields

# 특정 키 메모리 확인
redis-cli MEMORY USAGE user:data:massive

# 대형 해시 필드 수 확인
redis-cli HLEN user:data:massive
```

#### 지연 측정

```bash
# 실시간 레이턴시 측정
redis-cli --latency -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD>

# 레이턴시 히스토리 (1초 간격)
redis-cli --latency-history -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD>

# 고해상도 레이턴시 분포
redis-cli --latency-dist -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD>
```

#### 명령어 통계 분석

```bash
# 명령어별 호출 수 및 평균 처리 시간
redis-cli INFO commandstats | sort -t= -k2 -rn | head -20

# 출력 예시:
# cmdstat_get:calls=1000000,usec=2000000,usec_per_call=2.00
# cmdstat_hgetall:calls=5000,usec=500000,usec_per_call=100.00  ← 주목
```

### 2.3 Best Practice

- 운영 환경에서 `slowlog-log-slower-than 1000` (1ms) 으로 낮게 설정
- Slow Log는 정기적으로(`SLOWLOG GET` + 모니터링) 확인 후 `SLOWLOG RESET`
- O(N) 명령어는 데이터가 적을 때는 괜찮지만 성장 후 문제 → 사전에 SCAN 패턴 적용
- 대형 키는 분할 저장 또는 만료 TTL 적용으로 관리

---

## 3. 트러블슈팅

### 3.1 KEYS * 명령어로 Redis 블로킹

#### 증상
- 특정 시점에 모든 Redis 요청이 수 초간 응답 없음
- Slow Log에 `KEYS *` 등장

#### 원인
- `KEYS *`는 싱글 스레드 Redis에서 전체 키 순회 → 다른 요청 블로킹

#### 해결 방법
```bash
# SCAN으로 즉시 교체 (애플리케이션 코드 수정 필요)
# KEYS * → SCAN 0 MATCH * COUNT 100

# 임시 조치: 해당 클라이언트 연결 종료
redis-cli CLIENT KILL ADDR <client-ip>:<port>
```

### 3.2 AOF fsync 지연

#### 증상
- Slow Log에 `SET`, `HSET` 등 쓰기 명령어가 반복 등장
- `appendfsync always` 설정 환경

#### 원인
- `appendfsync always`: 매 명령마다 디스크 fsync → I/O 병목

#### 해결 방법
```bash
# appendfsync를 everysec으로 변경 (1초마다 fsync)
redis-cli CONFIG SET appendfsync everysec

# 디스크 I/O 상태 확인
iostat -x 1 5
```

### 3.3 대형 Hash HGETALL 지연

#### 증상
- Slow Log에 `HGETALL` 반복 등장
- 특정 Hash 키 조회 시 수십 ms 지연

#### 원인
- Hash 필드 수가 수만 개 이상으로 증가

#### 해결 방법
```bash
# Hash 필드 수 확인
redis-cli HLEN user:large-data

# HSCAN으로 점진적 조회로 교체
redis-cli HSCAN user:large-data 0 COUNT 100

# 구조 재설계: 하나의 큰 Hash → 여러 개 작은 Hash
# user:data:{userId}:profile
# user:data:{userId}:preferences
# user:data:{userId}:stats
```

---

## 4. 모니터링 및 확인

```bash
# Slow Log 정기 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> SLOWLOG GET 20

# Slow Log 항목 수 (0이 이상적)
redis-cli SLOWLOG LEN

# 명령어 통계 (평균 처리 시간 기준 정렬)
redis-cli INFO commandstats | awk -F'[=,]' '{print $8, $1}' | sort -rn | head -10
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_slowlog_length` | Slow Log 항목 수 | > 0 지속 시 |
| `redis_slowlog_last_id` | 마지막 Slow Log ID | 빠르게 증가 시 |
| `redis_commands_duration_seconds_total` | 명령어 처리 총 시간 | 급격한 증가 |

---

## 5. TIP

- `redis-cli --pipe` 모드로 다중 명령어를 배치 전송 → 왕복 지연 절감
- Pipeline(`MULTI/EXEC`)으로 여러 명령어를 단일 왕복에 처리
- `DEBUG SLEEP 0` 명령으로 지연 주입 테스트 가능
- 참고: [Redis Latency Monitoring](https://redis.io/docs/management/optimization/latency/)
