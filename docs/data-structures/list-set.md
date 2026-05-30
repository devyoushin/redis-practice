# List, Set 자료구조

Redis의 List(리스트)는 순서가 있는 요소 컬렉션으로 큐(Queue)와 스택(Stack) 구현에 사용합니다. Set(집합)은 순서 없는 고유 요소 컬렉션으로 중복 제거, 교집합/합집합 연산에 활용합니다.

---

## 1. 개요

| 자료구조 | 특성 | 주요 사용 사례 |
|---------|------|-------------|
| List | 순서 있음, 중복 허용, 양방향 삽입/삭제 | 메시지 큐, 작업 목록, 최근 항목 |
| Set | 순서 없음, 중복 불허, 집합 연산 지원 | 태그, 팔로워/팔로잉, 방문자 추적 |

---

## 2. 설명

### 2.1 핵심 개념

#### List 주요 명령어

| 명령어 | 시간복잡도 | 설명 |
|--------|-----------|------|
| `LPUSH key value` | O(1) | 왼쪽(Head)에 삽입 |
| `RPUSH key value` | O(1) | 오른쪽(Tail)에 삽입 |
| `LPOP key` | O(1) | 왼쪽에서 꺼냄 |
| `RPOP key` | O(1) | 오른쪽에서 꺼냄 |
| `BLPOP key timeout` | O(1) | 블로킹 LPOP (큐 패턴) |
| `BRPOP key timeout` | O(1) | 블로킹 RPOP |
| `LRANGE key start stop` | O(N) | 범위 조회 |
| `LLEN key` | O(1) | 리스트 길이 |
| `LTRIM key start stop` | O(N) | 범위 외 요소 삭제 |

#### List 사용 패턴

```
Queue(큐): LPUSH → RPOP  (FIFO)
Stack(스택): LPUSH → LPOP (LIFO)
최근 N개: LPUSH + LTRIM 0 N-1
```

#### Set 주요 명령어

| 명령어 | 시간복잡도 | 설명 |
|--------|-----------|------|
| `SADD key member` | O(1) | 멤버 추가 |
| `SREM key member` | O(1) | 멤버 삭제 |
| `SMEMBERS key` | O(N) | 전체 멤버 조회 |
| `SISMEMBER key member` | O(1) | 멤버 존재 여부 |
| `SMISMEMBER key m1 m2` | O(N) | 다중 멤버 존재 여부 |
| `SCARD key` | O(1) | 멤버 수 |
| `SUNION k1 k2` | O(N) | 합집합 |
| `SINTER k1 k2` | O(N×M) | 교집합 |
| `SDIFF k1 k2` | O(N) | 차집합 |
| `SRANDMEMBER key n` | O(N) | 랜덤 N개 조회 |
| `SPOP key` | O(1) | 랜덤 멤버 꺼냄 |
| `SSCAN key cursor` | O(1) | 점진적 순회 |

### 2.2 실무 적용 코드

#### List — 작업 큐 (Task Queue)

```redis-cli
# 작업 추가 (Producer)
RPUSH task:queue:image-resize '{"jobId":"job-001","imageUrl":"s3://..."}'
RPUSH task:queue:image-resize '{"jobId":"job-002","imageUrl":"s3://..."}'

# 작업 꺼내기 (Consumer — 블로킹 대기)
BLPOP task:queue:image-resize 30   # 30초 대기

# 큐 깊이 확인
LLEN task:queue:image-resize

# 큐 내용 확인 (앞 10개)
LRANGE task:queue:image-resize 0 9
```

#### List — 최근 방문 기록 (N개 유지)

```redis-cli
# 최근 방문 상품 추가 (최신이 앞에 오도록)
LPUSH user:history:userId123 productId789
LTRIM user:history:userId123 0 9   # 최근 10개만 유지

# 방문 기록 조회
LRANGE user:history:userId123 0 -1
```

#### List — 알림 큐 (Notification Queue)

```redis-cli
# 알림 추가
LPUSH notification:queue:userId123 '{"type":"order_complete","orderId":"ORD-001"}'

# 알림 읽기 (페이지별)
LRANGE notification:queue:userId123 0 19   # 최신 20개

# 읽은 알림 삭제
LTRIM notification:queue:userId123 0 99   # 최근 100개만 유지
```

#### Set — 팔로워/팔로잉

```redis-cli
# 팔로우
SADD following:userId123 userId456
SADD followers:userId456 userId123

# 맞팔로우 확인 (교집합)
SINTER following:userId123 followers:userId123

# 팔로우 취소
SREM following:userId123 userId456

# 팔로잉 수
SCARD following:userId123

# 팔로잉 목록
SMEMBERS following:userId123
```

#### Set — 중복 방문자 제거

```redis-cli
# 오늘의 유니크 방문자 추가
SADD visitors:unique:20260430 userId123
SADD visitors:unique:20260430 userId456
SADD visitors:unique:20260430 userId123   # 중복 — 추가 안 됨

# 유니크 방문자 수
SCARD visitors:unique:20260430

# TTL 설정 (하루가 지나면 삭제)
EXPIRE visitors:unique:20260430 86400
```

#### Set — 공통 태그 (교집합)

```redis-cli
# 상품 태그 추가
SADD product:tags:101 "스포츠" "아웃도어" "방수"
SADD product:tags:102 "스포츠" "러닝" "경량"

# 공통 태그 (교집합)
SINTER product:tags:101 product:tags:102
# → "스포츠"

# 전체 태그 (합집합)
SUNION product:tags:101 product:tags:102
```

### 2.3 Best Practice

- List를 메시지 큐로 사용 시 `BLPOP`으로 폴링 오버헤드 제거
- 대형 Set(`SMEMBERS` O(N))은 `SSCAN`으로 점진적 순회
- Set 멤버가 정수이고 512개 이하면 intset으로 메모리 효율 높음
- List가 길어지면 `LTRIM`으로 주기적 정리 (메모리 관리)

---

## 3. 트러블슈팅

### 3.1 List 큐가 쌓임 (Consumer 지연)

#### 증상
- `LLEN task:queue:*` 값이 계속 증가
- Consumer가 처리하지 못하고 있음

#### 원인
- Consumer 처리 속도 < Producer 생산 속도
- Consumer 장애로 큐에서 꺼내지 않음

#### 해결 방법
```redis-cli
# 큐 깊이 확인
LLEN task:queue:image-resize

# Consumer 수 확인 (애플리케이션 모니터링)
# Consumer 스케일 아웃 또는 처리 속도 최적화

# 임시 조치: 큐 비우기 (⚠️ 데이터 유실)
# DEL task:queue:image-resize
```

### 3.2 SMEMBERS 지연 (대형 Set)

#### 증상
- Slow Log에 `SMEMBERS` 반복 등장
- 특정 Set 조회 시 응답 지연

#### 원인
- Set 멤버 수가 수십만 개 이상으로 증가

#### 해결 방법
```redis-cli
# Set 크기 확인
SCARD large-set-key

# SSCAN으로 점진적 조회 (Slow Log 방지)
SSCAN large-set-key 0 COUNT 100

# 구조 재설계: 단일 대형 Set → 샤딩 (예: set:shard:{hash} 패턴)
```

### 3.3 List 아이템 중복 처리

#### 증상
- 큐에서 꺼낸 작업이 중복 처리됨

#### 원인
- Consumer가 `RPOP` 후 처리 실패 → 재시도 시 이미 꺼낸 항목 누락

#### 해결 방법
```redis-cli
# 신뢰성 있는 큐 패턴: RPOPLPUSH로 처리 중 목록 관리
RPOPLPUSH task:queue:pending task:queue:processing

# 처리 완료 시
LREM task:queue:processing 1 "{처리한 항목 값}"

# 처리 실패 시 재큐
RPOPLPUSH task:queue:processing task:queue:pending
```

---

## 4. 모니터링 및 확인

```bash
# List 길이 (큐 깊이)
redis-cli LLEN task:queue:image-resize

# Set 크기
redis-cli SCARD visitors:unique:20260430

# 전체 키 타입별 분포
redis-cli INFO keyspace
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_memory_used_bytes` | 메모리 사용량 | maxmemory의 85% |
| `redis_commands_processed_total` | 처리 명령어 수 | 급격한 감소 |
| `redis_slowlog_length` | Slow Log 항목 수 | > 0 지속 시 |

---

## 5. TIP

- `BLPOP` 타임아웃은 connection timeout보다 짧게 설정 (예: connection 60s, BLPOP 30s)
- Set의 `SUNIONSTORE`, `SINTERSTORE`, `SDIFFSTORE`로 결과를 새 키에 저장 가능
- Redis 7.0+에서 `LMPOP`, `BLMPOP`으로 여러 리스트에서 동시에 꺼내기 가능
- 참고: [Redis List Commands](https://redis.io/commands/#list), [Redis Set Commands](https://redis.io/commands/#set)
