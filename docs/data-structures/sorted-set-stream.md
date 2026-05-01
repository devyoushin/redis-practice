# Sorted Set, Stream 자료구조

Redis의 Sorted Set(정렬된 집합)은 Score(점수) 기반으로 자동 정렬되는 고유 요소 컬렉션으로 랭킹, 우선순위 큐에 사용합니다. Stream(스트림)은 Kafka와 유사한 로그 기반 메시지 스트리밍 자료구조입니다.

---

## 1. 개요

| 자료구조 | 특성 | 주요 사용 사례 |
|---------|------|-------------|
| Sorted Set | Score 기반 자동 정렬, 고유 멤버 | 랭킹 보드, 우선순위 큐, 시간 기반 필터링 |
| Stream | 추가 전용 로그, 소비자 그룹 | 이벤트 스트리밍, 감사 로그, 메시지 브로커 |

---

## 2. 설명

### 2.1 핵심 개념

#### Sorted Set 주요 명령어

| 명령어 | 시간복잡도 | 설명 |
|--------|-----------|------|
| `ZADD key score member` | O(log N) | 멤버 추가 (Score 포함) |
| `ZREM key member` | O(log N) | 멤버 삭제 |
| `ZSCORE key member` | O(1) | 멤버 Score 조회 |
| `ZRANK key member` | O(log N) | 멤버 오름차순 순위 (0부터) |
| `ZREVRANK key member` | O(log N) | 멤버 내림차순 순위 |
| `ZRANGE key start stop [WITHSCORES]` | O(log N+M) | 오름차순 범위 조회 |
| `ZREVRANGE key start stop` | O(log N+M) | 내림차순 범위 조회 |
| `ZRANGEBYSCORE key min max` | O(log N+M) | Score 범위 조회 |
| `ZINCRBY key increment member` | O(log N) | Score 증가 |
| `ZCARD key` | O(1) | 멤버 수 |
| `ZCOUNT key min max` | O(log N) | Score 범위 내 수 |

#### Stream 주요 명령어

| 명령어 | 설명 |
|--------|------|
| `XADD key * field value` | 메시지 추가 (`*`=자동 ID) |
| `XREAD COUNT n STREAMS key id` | 메시지 읽기 (단순) |
| `XREADGROUP GROUP g CONSUMER c STREAMS key >` | 소비자 그룹으로 읽기 |
| `XACK key group id` | 메시지 처리 완료 확인 |
| `XLEN key` | 스트림 길이 |
| `XRANGE key start end` | ID 범위 조회 |
| `XPENDING key group` | 미확인 메시지 조회 |
| `XGROUP CREATE key group id` | 소비자 그룹 생성 |
| `XTRIM key MAXLEN n` | 스트림 길이 제한 |

### 2.2 실무 적용 코드

#### Sorted Set — 실시간 랭킹 보드

```redis-cli
# 점수 추가/업데이트
ZADD ranking:game:daily 1500 "user:alice"
ZADD ranking:game:daily 2300 "user:bob"
ZADD ranking:game:daily 1800 "user:charlie"

# Top 10 조회 (내림차순, 점수 포함)
ZREVRANGE ranking:game:daily 0 9 WITHSCORES

# 특정 사용자 순위 조회
ZREVRANK ranking:game:daily "user:bob"   # 0 = 1위

# 점수 증가 (게임에서 점수 획득)
ZINCRBY ranking:game:daily 500 "user:alice"

# 특정 점수 이상인 사용자 수
ZCOUNT ranking:game:daily 2000 +inf

# 랭킹 초기화 (TTL 설정)
EXPIRE ranking:game:daily 86400   # 1일 후 만료
```

#### Sorted Set — 만료 예정 데이터 관리

```redis-cli
# 만료 시간을 Score로 사용 (Unix timestamp)
ZADD session:expire:set 1746100800 "session:abc123"   # 만료 시각 = Score
ZADD session:expire:set 1746187200 "session:def456"

# 현재 시간 이전(만료된) 세션 조회
ZRANGEBYSCORE session:expire:set 0 1746014400   # 현재 시각까지

# 만료된 세션 삭제
ZREMRANGEBYSCORE session:expire:set 0 1746014400
```

#### Sorted Set — 우선순위 큐

```redis-cli
# 우선순위별 작업 추가 (낮은 숫자 = 높은 우선순위)
ZADD job:queue:priority 1 '{"jobId":"critical-001","type":"payment"}'
ZADD job:queue:priority 5 '{"jobId":"normal-002","type":"email"}'
ZADD job:queue:priority 10 '{"jobId":"low-003","type":"report"}'

# 가장 높은 우선순위 작업 조회 후 처리 (원자적)
ZPOPMIN job:queue:priority 1
```

#### Stream — 이벤트 스트리밍

```redis-cli
# 이벤트 추가 (*=자동 생성 ID)
XADD order:events * orderId ORD-001 status created userId user123 amount 50000

# 최근 10개 이벤트 조회
XREVRANGE order:events + - COUNT 10

# 소비자 그룹 생성
XGROUP CREATE order:events order-processor $ MKSTREAM

# 소비자 그룹으로 메시지 읽기 (미처리 메시지)
XREADGROUP GROUP order-processor consumer-1 COUNT 10 STREAMS order:events >

# 처리 완료 확인 (ACK)
XACK order:events order-processor 1746014400000-0

# 미확인(처리 중) 메시지 확인
XPENDING order:events order-processor - + 10

# 스트림 크기 제한 (최근 10000개만 유지)
XTRIM order:events MAXLEN ~ 10000
```

### 2.3 Best Practice

- Sorted Set 랭킹은 일별/주별로 키 분리 (`ranking:game:daily:20260430`) + TTL 설정
- Stream Consumer Group은 서비스별 독립적으로 생성 — 여러 서비스가 동일 이벤트 소비 가능
- `XTRIM MAXLEN ~`(~는 approximate, 근사 절삭)으로 성능 오버헤드 없이 크기 제한
- Stream ACK 누락 시 `XPENDING`으로 미처리 메시지 재처리

---

## 3. 트러블슈팅

### 3.1 Sorted Set 랭킹 점수 오염

#### 증상
- 동일 사용자가 랭킹에 여러 번 등장

#### 원인
- Sorted Set은 멤버가 고유 — 발생 불가
- 멤버 문자열 불일치 (예: `user:123` vs `user123`)

#### 해결 방법
```redis-cli
# 중복 키 패턴 확인
ZRANGE ranking:game:daily 0 -1 WITHSCORES | grep "123"

# 잘못된 멤버 삭제
ZREM ranking:game:daily "user123"   # 잘못된 형식 삭제
```

### 3.2 Stream 미처리 메시지 누적 (PEL 증가)

#### 증상
- `XPENDING`으로 확인 시 미확인 메시지 계속 증가
- Consumer 장애 또는 ACK 누락

#### 원인
- Consumer가 메시지 읽은 후 처리 실패 → ACK 미호출

#### 해결 방법
```redis-cli
# 미확인 메시지 확인
XPENDING order:events order-processor - + 10

# 일정 시간 이상 미처리된 메시지 다른 Consumer에 재할당 (XCLAIM)
XCLAIM order:events order-processor consumer-2 60000 1746014400000-0
# 60000ms(1분) 이상 미처리된 메시지를 consumer-2로 재할당

# 처리 후 ACK
XACK order:events order-processor 1746014400000-0
```

### 3.3 Stream 무한 증가 (메모리 소모)

#### 증상
- Stream 키의 메모리 사용량이 지속 증가

#### 원인
- `XTRIM` 또는 `MAXLEN` 미설정

#### 해결 방법
```redis-cli
# 스트림 길이 확인
XLEN order:events

# 즉시 절삭 (최근 50000개만 유지)
XTRIM order:events MAXLEN 50000

# 이후 XADD에 MAXLEN 옵션 추가 (자동 관리)
XADD order:events MAXLEN ~ 50000 * orderId ORD-002 ...
```

---

## 4. 모니터링 및 확인

```bash
# Stream 길이 확인
redis-cli XLEN order:events

# Consumer Group 상태
redis-cli XINFO GROUPS order:events

# 소비자 정보
redis-cli XINFO CONSUMERS order:events order-processor

# Sorted Set 크기
redis-cli ZCARD ranking:game:daily
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_memory_used_bytes` | 메모리 사용량 | maxmemory의 85% |
| `redis_stream_length` | Stream 길이 (Custom) | 서비스별 임계값 |
| `redis_slowlog_length` | Slow Log 항목 수 | > 0 지속 시 |

---

## 5. TIP

- Redis 7.0+ `ZMPOP`, `BZMPOP`으로 여러 Sorted Set에서 동시에 Pop 가능
- Stream은 Kafka 대비 소규모 이벤트(초당 수만 건 이하)에 적합
- `XAUTOCLAIM`(Redis 6.2+)으로 일정 시간 미처리된 메시지 자동 재할당 가능
- Sorted Set Score는 64-bit double 정밀도 부동소수점 → 정수 Score 권장
- 참고: [Redis Sorted Set](https://redis.io/commands/#sorted_set), [Redis Streams](https://redis.io/docs/data-types/streams/)
