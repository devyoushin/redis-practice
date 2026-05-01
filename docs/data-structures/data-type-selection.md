# 자료구조 선택 가이드

Redis 자료구조 선택은 접근 패턴, 메모리 효율, 연산 복잡도를 고려하여 결정합니다. 잘못된 자료구조 선택은 성능 저하와 메모리 낭비로 이어집니다.

---

## 1. 개요

Redis 5가지 핵심 자료구조 비교:

| 자료구조 | 순서 | 중복 | 인덱스 | 최적 사용 사례 |
|---------|------|------|--------|-------------|
| String | - | - | 키 기반 | 단일 값, 카운터, 캐시 |
| Hash | - | 필드 고유 | 필드 기반 | 구조화된 객체 (사용자, 상품) |
| List | 삽입 순서 | 허용 | 인덱스 기반 | 큐, 스택, 최근 기록 |
| Set | 없음 | 불허 | 없음 | 고유 컬렉션, 집합 연산 |
| Sorted Set | Score 기반 | 멤버 고유 | Score/Rank | 랭킹, 우선순위 큐 |

---

## 2. 설명

### 2.1 핵심 개념

#### 사용 사례별 자료구조 선택

| 사용 사례 | 권장 자료구조 | 이유 |
|---------|------------|------|
| 단순 캐시 (JSON) | String | 직렬화/역직렬화 한 번에 처리 |
| 객체 부분 필드 접근 | Hash | 전체 역직렬화 없이 필드 조회 |
| 메시지 큐 (FIFO) | List | RPUSH + BLPOP으로 효율적 큐 |
| 중복 없는 방문자 수 | Set | SADD + SCARD O(1) |
| 실시간 랭킹 | Sorted Set | Score 기반 자동 정렬 |
| 이벤트 스트리밍 | Stream | 소비자 그룹, ACK 지원 |
| 비트 단위 카운터 | Bitmap (String) | 수억 개 플래그를 수MB로 관리 |
| 확률적 집합 크기 | HyperLogLog | 메모리 12KB로 수십억 고유값 추정 |

#### String vs Hash — 객체 저장 선택 기준

```
String 선택 시:
  ✅ 객체 전체를 항상 한 번에 읽고 쓰는 경우
  ✅ JSON 직렬화 라이브러리 이미 사용 중
  ✅ 객체 크기 가변 (필드 수 동적 변경)

Hash 선택 시:
  ✅ 객체 일부 필드만 자주 업데이트 (HSET field)
  ✅ 필드별 TTL이 아닌 전체 객체 TTL 관리
  ✅ 필드 수 128개 이하 + 값 64바이트 이하 (ziplist 메모리 효율)

Hash 비권장 시:
  ❌ 필드 수가 동적으로 수천 개 이상 증가하는 경우
  ❌ 필드 자체에 TTL 필요한 경우 (Hash는 필드별 TTL 미지원)
```

#### List vs Stream — 메시지 큐 선택 기준

```
List 선택 시:
  ✅ 단순 FIFO 큐
  ✅ 메시지 유실 허용 가능
  ✅ 소비자가 1개 또는 경쟁 소비 패턴
  ✅ 구현 단순성 우선

Stream 선택 시:
  ✅ 메시지 처리 보장 필요 (ACK 패턴)
  ✅ 여러 소비자 그룹이 동일 메시지 독립적으로 소비
  ✅ 메시지 히스토리 보존 필요
  ✅ 재처리 (Replay) 필요
```

#### Set vs Sorted Set

```
Set 선택 시:
  ✅ 순서 불필요
  ✅ 집합 연산 필요 (교집합, 합집합)
  ✅ 단순 고유 컬렉션

Sorted Set 선택 시:
  ✅ 순위/정렬 필요
  ✅ Score 범위 조회 필요
  ✅ 우선순위 큐
```

### 2.2 실무 적용 코드

#### 자료구조별 메모리 사용량 비교

```redis-cli
# 동일 데이터를 다른 자료구조로 저장 후 메모리 비교
# 사용자 정보 (id, name, email, age)

# String (JSON)
SET user:str:1 '{"id":1,"name":"홍길동","email":"hong@test.com","age":30}'
MEMORY USAGE user:str:1

# Hash
HSET user:hash:1 id 1 name "홍길동" email "hong@test.com" age 30
MEMORY USAGE user:hash:1

# 인코딩 확인 (ziplist vs hashtable)
OBJECT ENCODING user:hash:1
```

#### 적절한 자료구조 조합 예시

```redis-cli
# 쇼핑몰 예시: 여러 자료구조 조합
# 1. 상품 정보 캐시 (Hash — 필드별 접근 가능)
HSET product:1001 name "운동화" price 59000 stock 100 category "스포츠"
EXPIRE product:1001 3600

# 2. 실시간 인기 상품 랭킹 (Sorted Set — 조회수 기반)
ZINCRBY ranking:product:popular 1 "product:1001"
ZREVRANGE ranking:product:popular 0 9 WITHSCORES

# 3. 장바구니 (Hash — 사용자별)
HSET cart:userId123 product:1001 2 product:1002 1
HGETALL cart:userId123

# 4. 최근 본 상품 (List — 최근 10개)
LPUSH recently:viewed:userId123 product:1001
LTRIM recently:viewed:userId123 0 9

# 5. 오늘의 구매자 (Set — 중복 제거)
SADD buyers:today:20260430 userId123
SCARD buyers:today:20260430
```

### 2.3 Best Practice

- 자료구조 선택 전 **접근 패턴** 먼저 정의 (읽기 빈도, 쓰기 패턴, 조회 범위)
- 메모리 효율을 위해 소형 자료구조는 ziplist/intset 인코딩 유지 (임계값 내 유지)
- 하나의 키에 모든 데이터 집중 금지 — 큰 키는 조회/삭제 시 블로킹 발생

---

## 3. 트러블슈팅

### 3.1 자료구조 인코딩 전환으로 메모리 급증

#### 증상
- 특정 키의 메모리 사용량이 갑자기 크게 증가

#### 원인
- Hash/List/Set/Sorted Set이 임계값 초과 → ziplist에서 hashtable로 인코딩 전환
- 전환 시 메모리 사용량 2~10배 증가

#### 해결 방법
```redis-cli
# 현재 인코딩 확인
OBJECT ENCODING my-key

# Hash ziplist 임계값 설정 확인/조정
CONFIG GET hash-max-ziplist-entries   # 기본 128
CONFIG GET hash-max-ziplist-value     # 기본 64 (bytes)

# 대안: 데이터 분할 (하나의 큰 Hash → 여러 작은 Hash)
```

### 3.2 잘못된 자료구조로 인한 O(N) 블로킹

#### 증상
- 특정 명령어 실행 시 Redis 응답 없음 (블로킹)
- Slow Log에 `SMEMBERS`, `HGETALL`, `LRANGE 0 -1` 반복

#### 원인
- 대형 컬렉션에 O(N) 명령어 사용

#### 해결 방법
```redis-cli
# SMEMBERS → SSCAN으로 교체
SSCAN large-set 0 COUNT 100

# HGETALL → 필요한 필드만 HMGET
HMGET user:hash:1 name email

# LRANGE 0 -1 → 페이징 처리
LRANGE list-key 0 49   # 50개씩 페이징
```

---

## 4. 모니터링 및 확인

```bash
# 키 타입 확인
redis-cli TYPE my-key

# 인코딩 확인
redis-cli OBJECT ENCODING my-key

# 키 메모리 사용량
redis-cli MEMORY USAGE my-key

# 전체 키 통계
redis-cli INFO keyspace
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_memory_used_bytes` | 메모리 사용량 | maxmemory의 85% |
| `redis_slowlog_length` | Slow Log 항목 수 | > 0 지속 시 |

---

## 5. TIP

- `OBJECT ENCODING key`로 현재 인코딩 확인 — 성능 최적화 첫 단계
- HyperLogLog: 수십억 고유 요소 수를 12KB 메모리로 오차율 0.81% 내에서 추정
- Bitmap: 수억 개 boolean 플래그를 수MB로 관리 (DAU, 연속 출석 등)
- 참고: [Redis Data Types](https://redis.io/docs/data-types/)
