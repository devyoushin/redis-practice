# String, Hash 자료구조

Redis의 String(문자열)과 Hash(해시)는 가장 기본적인 자료구조입니다. String은 단일 값 저장에, Hash는 객체(Object) 형태의 구조화된 데이터 저장에 사용합니다.

---

## 1. 개요

| 자료구조 | 최대 크기 | 주요 사용 사례 |
|---------|---------|-------------|
| String | 512MB | 캐시, 카운터, 세션, 분산 락 |
| Hash | 필드 수 2^32-1 | 사용자 정보, 상품 정보, 설정값 |

---

## 2. 설명

### 2.1 핵심 개념

#### String 주요 명령어

| 명령어 | 시간복잡도 | 설명 |
|--------|-----------|------|
| `SET key value [EX seconds]` | O(1) | 값 설정 (TTL 포함 가능) |
| `GET key` | O(1) | 값 조회 |
| `INCR key` | O(1) | 정수값 1 증가 (카운터) |
| `INCRBY key n` | O(1) | 정수값 N 증가 |
| `SETNX key value` | O(1) | 키 없을 때만 설정 (분산 락) |
| `GETSET key value` | O(1) | 이전 값 반환 후 새 값 설정 |
| `MSET k1 v1 k2 v2` | O(N) | 다중 키 일괄 설정 |
| `MGET k1 k2` | O(N) | 다중 키 일괄 조회 |

#### Hash 주요 명령어

| 명령어 | 시간복잡도 | 설명 |
|--------|-----------|------|
| `HSET key field value` | O(1) | 필드 설정 |
| `HGET key field` | O(1) | 필드 조회 |
| `HMSET key f1 v1 f2 v2` | O(N) | 다중 필드 설정 |
| `HMGET key f1 f2` | O(N) | 다중 필드 조회 |
| `HGETALL key` | O(N) | 전체 필드/값 조회 |
| `HKEYS key` | O(N) | 필드 목록 |
| `HVALS key` | O(N) | 값 목록 |
| `HDEL key field` | O(1) | 필드 삭제 |
| `HEXISTS key field` | O(1) | 필드 존재 여부 |
| `HINCRBY key field n` | O(1) | 필드 정수값 증가 |

#### TTL(Time To Live) 관리

```
SET key value EX 3600          → 1시간 후 만료
EXPIRE key 3600                → 기존 키에 TTL 설정
TTL key                        → 남은 TTL(초) 조회 (-1: 만료없음, -2: 키없음)
PERSIST key                    → TTL 제거 (영구 저장)
```

### 2.2 실무 적용 코드

#### String — 캐시 패턴 (Cache-Aside)

```redis-cli
# 상품 정보 캐시 (JSON 문자열, TTL 1시간)
SET product:cache:12345 '{"id":12345,"name":"상품명","price":10000}' EX 3600

# 조회
GET product:cache:12345

# 존재 여부 확인
EXISTS product:cache:12345
```

#### String — 카운터 (Rate Limiting)

```redis-cli
# 사용자별 1분 요청 카운터
INCR rate:limit:userId123:20260430
EXPIRE rate:limit:userId123:20260430 60

# 현재 카운터 조회
GET rate:limit:userId123:20260430

# 카운터가 100 초과 시 요청 차단 (애플리케이션 로직)
```

#### String — 분산 락 (Distributed Lock)

```redis-cli
# 락 획득 (SET NX EX: 키 없을 때만 설정 + TTL)
SET lock:order:process 1 NX EX 30

# 락 해제 (소유자 확인 후 삭제 — Lua Script 사용 권장)
DEL lock:order:process
```

#### Hash — 사용자 세션 저장

```redis-cli
# 세션 저장 (Hash로 필드별 관리)
HSET session:user:abc123 userId 123 username "홍길동" role "admin" loginAt "2026-04-30T10:00:00"
EXPIRE session:user:abc123 1800   # 30분 TTL

# 특정 필드 조회
HGET session:user:abc123 username

# 전체 세션 조회
HGETALL session:user:abc123

# 필드 업데이트
HSET session:user:abc123 lastActiveAt "2026-04-30T10:15:00"
```

#### Hash — 설정값 관리

```redis-cli
# 서비스 설정값 Hash로 관리
HSET config:payment:service maxRetry 3 timeout 5000 currency KRW

# 단일 설정 조회
HGET config:payment:service maxRetry

# 전체 설정 조회
HGETALL config:payment:service

# 설정 업데이트
HSET config:payment:service timeout 3000
```

#### MGET/MSET — 배치 처리

```redis-cli
# 여러 상품 캐시 일괄 조회 (N번 왕복 vs 1번 왕복)
MGET product:cache:1 product:cache:2 product:cache:3

# 일괄 설정
MSET user:name:1 "홍길동" user:name:2 "김철수" user:name:3 "이영희"
```

### 2.3 Best Practice

- Hash의 `HGETALL`은 필드 수가 많으면 O(N) — 필요한 필드만 `HMGET`으로 조회
- String으로 JSON 저장 vs Hash 사용: 일부 필드만 접근 시 Hash, 전체 직렬화 시 String
- 분산 락은 `SET NX EX` + Lua Script로 원자적 해제 구현 (Redisson 라이브러리 권장)
- 카운터 키에는 반드시 TTL 설정 (메모리 누수 방지)

---

## 3. 트러블슈팅

### 3.1 TTL 없는 키 메모리 누수

#### 증상
- Redis 메모리 사용량이 지속 증가
- `INFO keyspace`에서 특정 DB의 키 수가 계속 증가

#### 원인
- 애플리케이션에서 `SET` 후 `EXPIRE` 설정 누락
- `MSET` 사용 시 TTL 설정 불가 → 별도 `EXPIRE` 호출 누락

#### 해결 방법
```redis-cli
# TTL 없는 키 확인 (SCAN으로 샘플링)
SCAN 0 MATCH "product:cache:*" COUNT 100
TTL product:cache:12345   # -1이면 TTL 없음

# 일괄 TTL 설정 (특정 패턴의 키)
# redis-cli --scan --pattern "product:cache:*" | xargs -I {} redis-cli EXPIRE {} 3600
```

### 3.2 Hash HGETALL 지연

#### 증상
- 특정 Hash 키 조회 시 응답 지연
- Slow Log에 `HGETALL` 반복 등장

#### 원인
- Hash 필드 수가 수천~수만 개 수준으로 증가

#### 해결 방법
```redis-cli
# Hash 필드 수 확인
HLEN user:data:large-hash

# HSCAN으로 점진적 조회
HSCAN user:data:large-hash 0 COUNT 100

# 구조 재설계: 하나의 큰 Hash → 여러 개의 작은 Hash로 분리
```

### 3.3 분산 락 해제 실패 (락 만료 전 처리 완료)

#### 증상
- 처리 완료 후 락 해제 시도 시 `DEL` 명령이 다른 프로세스의 락을 삭제

#### 원인
- 락 소유자 확인 없이 `DEL` 직접 호출 → Race Condition

#### 해결 방법
```redis-cli
# 올바른 방법: 락 값으로 소유자 확인 후 삭제 (Lua Script)
# SET lock:key {unique-token} NX EX 30
# Lua: if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) end
```

---

## 4. 모니터링 및 확인

```bash
# DB별 키 수 및 만료 키 통계
redis-cli INFO keyspace

# 메모리 사용량 확인
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"

# 특정 키 메모리 사용량
redis-cli MEMORY USAGE product:cache:12345

# TTL 없는 키 샘플링 확인
redis-cli SCAN 0 MATCH "*" COUNT 100
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_memory_used_bytes` | 메모리 사용량 | maxmemory의 85% |
| `redis_keyspace_hits_total` | 캐시 히트 | Hit Rate < 80% |
| `redis_keyspace_misses_total` | 캐시 미스 | Hit Rate 계산용 |
| `redis_expired_keys_total` | 만료된 키 수 | 급증 시 TTL 설계 검토 |

---

## 5. TIP

- `SET key value GET` (Redis 6.2+): 이전 값 반환 + 새 값 설정 원자적 처리
- Hash는 필드 수 128개 이하, 각 값 64바이트 이하면 ziplist로 메모리 효율 높음
- `OBJECT ENCODING key`로 현재 인코딩 확인 가능 (ziplist vs hashtable)
- 참고: [Redis String Commands](https://redis.io/commands/#string), [Redis Hash Commands](https://redis.io/commands/#hash)
