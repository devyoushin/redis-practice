# AUTH 인증 및 ACL

Redis 6.0+에서 ACL(Access Control List)이 도입되어 사용자별 명령어/키 접근 제어가 가능합니다. 이전 방식인 `requirepass`는 단일 전역 비밀번호로 ACL의 `default` 사용자와 동일합니다.

---

## 1. 개요

Redis 인증 방식:

| 방식 | Redis 버전 | 설명 | 권장 여부 |
|------|-----------|------|---------|
| `requirepass` | 모든 버전 | 단일 비밀번호 | 단순 환경 |
| ACL | 6.0+ | 사용자별 권한, 키 패턴 제한 | **권장** |

---

## 2. 설명

### 2.1 핵심 개념

#### ACL 규칙 문법

```
ACL SETUSER <username> [규칙...]

규칙:
  on/off               → 사용자 활성화/비활성화
  ><password>          → 비밀번호 설정 (>는 접두사)
  ~<pattern>           → 키 패턴 접근 허용 (~* = 모든 키)
  &<pattern>           → Pub/Sub 채널 패턴
  +<command>           → 명령어 허용 (+@all = 모든 명령어)
  -<command>           → 명령어 거부
  +@<category>         → 명령어 카테고리 허용
  nopass               → 비밀번호 없음 (주의)
  resetkeys            → 키 접근 초기화
  reset                → 모든 규칙 초기화
```

#### 명령어 카테고리 주요 목록

| 카테고리 | 포함 명령어 |
|---------|-----------|
| `@read` | GET, HGET, LRANGE, SMEMBERS 등 |
| `@write` | SET, HSET, LPUSH, SADD 등 |
| `@string` | String 자료구조 명령어 |
| `@hash` | Hash 자료구조 명령어 |
| `@admin` | CONFIG, DEBUG, FLUSHALL 등 (위험) |
| `@dangerous` | KEYS, FLUSHDB, MIGRATE 등 |
| `@all` | 모든 명령어 |

### 2.2 실무 적용 코드

#### requirepass 설정 (단순 환경)

```
# redis.conf
requirepass <REDIS_PASSWORD>
```

```bash
# 인증 후 접속
redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a <REDIS_PASSWORD>

# 또는 접속 후 AUTH 명령어
redis-cli -h redis-master.redis.svc.cluster.local -p 6379
AUTH <REDIS_PASSWORD>
```

#### ACL 사용자 생성 및 관리

```bash
# 현재 ACL 목록 확인
redis-cli -a <ADMIN_PASSWORD> ACL LIST

# 읽기 전용 사용자 생성 (캐시 조회 서비스용)
redis-cli -a <ADMIN_PASSWORD> ACL SETUSER readonly-user on >readonly-password ~* +@read

# 특정 키 패턴만 접근 가능한 사용자 (payment 서비스용)
redis-cli -a <ADMIN_PASSWORD> ACL SETUSER payment-service on >payment-password \
  ~payment:* ~rate:limit:* +@read +@write -@dangerous

# 관리자 사용자 (전체 권한)
redis-cli -a <ADMIN_PASSWORD> ACL SETUSER admin-user on >admin-password ~* +@all

# 사용자 정보 확인
redis-cli -a <ADMIN_PASSWORD> ACL GETUSER payment-service

# 사용자 비활성화
redis-cli -a <ADMIN_PASSWORD> ACL SETUSER readonly-user off

# 사용자 삭제
redis-cli -a <ADMIN_PASSWORD> ACL DELUSER readonly-user
```

#### ACL 파일로 관리 (권장)

```
# /etc/redis/users.acl

# 기본 사용자 비활성화
user default off nopass nocommands nokeys

# 관리자 계정
user admin on >admin-secure-password ~* +@all

# 캐시 서비스 계정 (읽기/쓰기, 위험 명령어 제외)
user cache-service on >cache-password ~cache:* +@read +@write -@dangerous -@admin

# 읽기 전용 계정
user readonly on >readonly-password ~* +@read -@dangerous
```

```
# redis.conf
aclfile /etc/redis/users.acl
```

```bash
# ACL 파일 변경 후 리로드
redis-cli -a <ADMIN_PASSWORD> ACL LOAD

# 현재 ACL을 파일로 저장
redis-cli -a <ADMIN_PASSWORD> ACL SAVE
```

#### Kubernetes Secret으로 비밀번호 관리

```yaml
# redis-password-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-auth
  namespace: redis
type: Opaque
stringData:
  redis-password: "<REDIS_PASSWORD>"
  admin-password: "<ADMIN_PASSWORD>"
  cache-service-password: "<CACHE_PASSWORD>"
```

```yaml
# Deployment에서 Secret 참조
env:
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-auth
        key: cache-service-password
```

#### Bitnami Helm ACL 설정

```yaml
# redis-sentinel-values.yaml
auth:
  enabled: true
  password: ""              # Kubernetes Secret 관리

# ACL 사용자 추가 설정은 redis.config로 관리
master:
  config: |
    aclfile /opt/bitnami/redis/mounted-etc/users.acl
```

### 2.3 Best Practice

- 서비스별 전용 ACL 사용자 생성 — 단일 비밀번호 공유 금지
- `default` 사용자 비활성화 또는 최소 권한만 부여
- `@dangerous` 카테고리 (`KEYS`, `FLUSHDB` 등) 서비스 계정에 부여 금지
- ACL 사용자 목록을 ACL 파일(`aclfile`)로 버전 관리

---

## 3. 트러블슈팅

### 3.1 WRONGPASS 인증 오류

#### 증상
```
WRONGPASS invalid username-password pair or user is disabled.
```

#### 원인
- 잘못된 비밀번호 또는 사용자 비활성화

#### 해결 방법
```bash
# 사용자 상태 확인
redis-cli -a <ADMIN_PASSWORD> ACL GETUSER cache-service

# 비밀번호 재설정
redis-cli -a <ADMIN_PASSWORD> ACL SETUSER cache-service >new-password

# 사용자 활성화
redis-cli -a <ADMIN_PASSWORD> ACL SETUSER cache-service on
```

### 3.2 NOPERM 명령어 권한 오류

#### 증상
```
NOPERM this user has no permissions to run the 'keys' command
```

#### 원인
- 사용자 ACL에 해당 명령어 권한 없음

#### 해결 방법
```bash
# 현재 권한 확인
redis-cli -a <ADMIN_PASSWORD> ACL GETUSER cache-service

# 필요한 명령어 권한 추가 (SCAN 허용, KEYS 계속 금지)
redis-cli -a <ADMIN_PASSWORD> ACL SETUSER cache-service +scan

# 또는 해당 서비스 코드에서 KEYS → SCAN으로 교체
```

### 3.3 ACL 파일 로드 실패

#### 증상
```
ERR Error loading ACL file: /etc/redis/users.acl: No such file or directory
```

#### 원인
- ACL 파일 경로 오류 또는 파일 없음

#### 해결 방법
```bash
# ACL 파일 경로 확인
redis-cli CONFIG GET aclfile

# ACL 파일 생성 (현재 ACL 기준)
redis-cli -a <ADMIN_PASSWORD> ACL SAVE

# 파일 존재 확인
kubectl exec -it redis-master-0 -n redis -- ls -la /etc/redis/
```

---

## 4. 모니터링 및 확인

```bash
# 현재 ACL 사용자 목록
redis-cli -a <ADMIN_PASSWORD> ACL LIST

# 특정 사용자 권한 확인
redis-cli -a <ADMIN_PASSWORD> ACL GETUSER cache-service

# ACL 로그 (인증 실패 기록)
redis-cli -a <ADMIN_PASSWORD> ACL LOG

# ACL 로그 초기화
redis-cli -a <ADMIN_PASSWORD> ACL LOG RESET
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_acl_access_denied_auth` | 인증 실패 수 | > 0 급증 시 |
| `redis_acl_access_denied_cmd` | 명령어 권한 거부 수 | > 0 |
| `redis_acl_access_denied_key` | 키 접근 거부 수 | > 0 |

---

## 5. TIP

- Redis 7.0+에서 ACL `%R~pattern` (읽기 전용 키 패턴), `%W~pattern` (쓰기 전용 키 패턴) 지원
- `ACL WHOAMI`로 현재 인증된 사용자 확인
- Bitnami Redis는 기본적으로 `requirepass` 방식 — ACL 사용 시 별도 설정 필요
- 참고: [Redis ACL](https://redis.io/docs/management/security/acl/)
