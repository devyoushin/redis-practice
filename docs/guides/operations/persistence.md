# Redis 영속성 (Persistence)

Redis 영속성(Persistence)은 메모리 데이터를 디스크에 저장하여 재시작 후에도 데이터를 복구하는 기능입니다. RDB(Redis Database)와 AOF(Append Only File) 두 방식이 있으며, 운영 환경에서는 두 가지를 병행 사용합니다.

---

## 1. 개요

| 방식 | 설명 | 장점 | 단점 |
|------|------|------|------|
| RDB | 특정 시점 스냅샷 저장 | 파일 크기 작음, 빠른 복구 | 저장 간격 사이 데이터 유실 가능 |
| AOF | 모든 쓰기 명령 로그 | 데이터 유실 최소화 | 파일 크기 큼, 복구 느림 |
| RDB + AOF | 두 방식 병행 | 안정성 + 효율성 | 디스크 사용량 증가 |

---

## 2. 설명

### 2.1 핵심 개념

#### RDB(Redis Database Snapshot)

```
동작:
  1. BGSAVE 명령 또는 save 조건 충족
  2. fork() → 자식 프로세스가 스냅샷 저장
  3. 부모 프로세스는 계속 요청 처리
  4. dump.rdb 파일로 저장 완료

save 조건 (redis.conf):
  save 900 1      → 900초 내 1개 이상 변경 시 저장
  save 300 10     → 300초 내 10개 이상 변경 시 저장
  save 60 10000   → 60초 내 10000개 이상 변경 시 저장
```

#### AOF(Append Only File)

```
appendfsync 옵션:
  always    → 매 명령마다 fsync (안전, 느림)
  everysec  → 1초마다 fsync (균형, 권장)
  no        → OS에 위임 (빠름, 유실 위험)

AOF Rewrite:
  → 누적된 AOF 파일을 현재 메모리 상태로 재생성
  → auto-aof-rewrite-percentage 100 (파일 크기 2배 시 자동 실행)
  → auto-aof-rewrite-min-size 64mb (최소 64MB 이상일 때만)
```

#### 복구 우선순위

```
AOF 파일 있음 → AOF로 복구 (더 최신 데이터)
AOF 없음      → RDB로 복구
둘 다 없음    → 빈 상태로 시작
```

### 2.2 실무 적용 코드

#### 운영 환경 권장 설정 (redis.conf)

```
# RDB 설정
save 900 1
save 300 10
save 60 10000
rdbcompression yes              # RDB 파일 LZF 압축
rdbchecksum yes                 # RDB 파일 CRC64 체크섬
dbfilename dump.rdb
dir /data

# AOF 설정
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec            # 1초마다 fsync (성능/안정성 균형)
no-appendfsync-on-rewrite no    # Rewrite 중에도 fsync 유지 (안전)
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes        # AOF 파일 앞부분에 RDB 포함 (빠른 로딩)
```

#### RDB/AOF 상태 확인

```bash
# 영속성 상태 전체 확인
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> INFO persistence

# 출력 예시:
# rdb_last_save_time:1746014400
# rdb_last_bgsave_status:ok
# aof_enabled:1
# aof_rewrite_in_progress:0
# aof_last_bgrewrite_status:ok
# aof_current_size:104857600    # 현재 AOF 크기 (100MB)
```

#### 수동 RDB 저장

```bash
# 비동기 RDB 저장 (BGSAVE — 자식 프로세스 생성)
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> BGSAVE

# 동기 RDB 저장 (SAVE — 블로킹, 운영 환경 주의)
# redis-cli SAVE  ← 운영 환경 사용 금지

# 마지막 저장 시각 확인
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> LASTSAVE
```

#### AOF Rewrite 수동 실행

```bash
# AOF 파일 크기 줄이기
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> BGREWRITEAOF

# AOF Rewrite 완료 여부 확인
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> INFO persistence | grep aof_rewrite_in_progress
```

#### RDB 파일로 복구 절차

```bash
# 1. Redis 중지
kubectl scale deployment redis -n redis --replicas=0

# 2. 기존 RDB 파일 백업
kubectl exec -it redis-master-0 -n redis -- ls /data/
# dump.rdb 파일 확인

# 3. 복구할 RDB 파일 복사
kubectl cp /local/path/dump.rdb redis/redis-master-0:/data/dump.rdb

# 4. Redis 재시작
kubectl scale deployment redis -n redis --replicas=1

# 5. 복구 확인
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a <PASSWORD> DBSIZE
```

#### Bitnami Helm 영속성 설정

```yaml
# redis-sentinel-values.yaml
master:
  persistence:
    enabled: true
    storageClass: gp3
    size: 10Gi
    path: /data

  config: |
    save 900 1
    save 300 10
    appendonly yes
    appendfsync everysec
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    aof-use-rdb-preamble yes
```

### 2.3 Best Practice

- 운영 환경에서 `appendonly yes` + `appendfsync everysec` 조합 권장
- `aof-use-rdb-preamble yes`로 AOF 로딩 속도 개선 (기본값 yes)
- Redis Cluster 모드에서는 RDB만 사용하는 경우도 많음 (AOF Rewrite 부하 분산)
- 정기 RDB 백업: `BGSAVE` 후 PVC에서 외부 스토리지(S3)로 복사

---

## 3. 트러블슈팅

### 3.1 BGSAVE 실패 (fork 오류)

#### 증상
```
Can't save in background: fork: Cannot allocate memory
```

#### 원인
- 시스템 메모리 부족으로 fork() 실패
- Linux `vm.overcommit_memory=0` 설정 시 메모리 예측 실패

#### 해결 방법
```bash
# Linux overcommit_memory 설정 (Kubernetes DaemonSet으로 적용)
sysctl -w vm.overcommit_memory=1

# 또는 Redis 설정에서 RDB 비활성화 후 AOF만 사용
# redis-cli CONFIG SET save ""
```

### 3.2 AOF 파일 손상 (복구 오류)

#### 증상
```
* Unrecoverable error in append only file: make sure Redis is not running or there are no -rw- permissions on the file
Bad file format reading the append only file: make it writable or use redis-check-aof
```

#### 원인
- Redis 비정상 종료 중 AOF 파일 마지막 명령이 불완전하게 기록

#### 해결 방법
```bash
# AOF 파일 검사 및 복구
redis-check-aof --fix /data/appendonly.aof

# 복구 후 Redis 재시작
kubectl delete pod redis-master-0 -n redis
```

### 3.3 RDB 저장 중 성능 저하

#### 증상
- `BGSAVE` 실행 중 응답 지연 증가
- fork() 시 메모리 복사로 인한 일시적 지연

#### 원인
- 대용량 메모리 + Copy-on-Write 오버헤드
- 자식 프로세스가 디스크 I/O 집중

#### 해결 방법
```bash
# RDB 저장 빈도 줄이기
redis-cli CONFIG SET save "3600 1"   # 1시간마다 1개 변경 시

# 또는 RDB 비활성화 + AOF만 사용
redis-cli CONFIG SET save ""
```

---

## 4. 모니터링 및 확인

```bash
# 영속성 상태 확인
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <PASSWORD> INFO persistence

# RDB 마지막 저장 시간 (Unix timestamp)
redis-cli LASTSAVE

# AOF 파일 크기 확인
kubectl exec -it redis-master-0 -n redis -- ls -lh /data/
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_rdb_last_bgsave_status` | RDB 마지막 저장 성공 여부 (ok=1) | != 1 |
| `redis_aof_enabled` | AOF 활성화 여부 | 운영환경 0이면 경고 |
| `redis_aof_rewrite_in_progress` | AOF Rewrite 진행 중 | 장시간 지속 시 |
| `redis_rdb_last_save_timestamp_seconds` | 마지막 RDB 저장 시각 | 현재 시각과 차이 > 1시간 |

---

## 5. TIP

- `redis-check-rdb dump.rdb`로 RDB 파일 무결성 사전 확인
- AOF와 RDB 모두 활성화 시 복구 시 AOF 우선 사용 (`aof-use-rdb-preamble yes` 제외)
- Redis 7.0+ `OBJECT FREQ`로 LFU 접근 빈도 확인 가능
- 참고: [Redis Persistence](https://redis.io/docs/management/persistence/)
