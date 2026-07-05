# Redis systemd 설치

운영 환경에서 가장 일반적인 방식입니다. Redis 프로세스를 OS 서비스로 관리하고, 디스크/메모리/커널 설정을 서버 단위로 통제할 수 있습니다.

## 구성 개요

| 항목 | 값 |
|------|-----|
| 실행 사용자 | `redis` |
| 설정 파일 | `/etc/redis/redis.conf` |
| 데이터 디렉터리 | `/var/lib/redis` |
| 로그 디렉터리 | `/var/log/redis` |
| systemd unit | `/etc/systemd/system/redis.service` |

`ops/01-installation/redis.service`는 source 설치 기준으로 `/usr/local/bin/redis-server`를 사용합니다. RPM/DEB 패키지 설치를 사용한다면 배포판의 기본 unit을 쓰거나 `ExecStart` 경로를 `/usr/bin/redis-server`로 조정합니다.

## 설치 흐름

```bash
sudo useradd --system --home /var/lib/redis --shell /sbin/nologin redis
sudo mkdir -p /etc/redis /var/lib/redis /var/log/redis
sudo chown -R redis:redis /var/lib/redis /var/log/redis
sudo chmod 750 /var/lib/redis
```

`ops/config/redis.conf`를 기준으로 환경에 맞게 수정한 뒤 배치합니다.

```bash
sudo cp ops/config/redis.conf /etc/redis/redis.conf
sudo chown redis:redis /etc/redis/redis.conf
sudo chmod 640 /etc/redis/redis.conf
```

`ops/01-installation/redis.service`를 systemd unit으로 등록합니다.

```bash
sudo cp ops/01-installation/redis.service /etc/systemd/system/redis.service
sudo systemctl daemon-reload
sudo systemctl enable --now redis
```

## 확인

```bash
systemctl status redis
redis-cli -a '<REDIS_PASSWORD>' ping
redis-cli -a '<REDIS_PASSWORD>' INFO server
```

## 운영 체크

- `bind`는 운영 접근망에 맞게 제한합니다.
- `requirepass` 또는 ACL을 반드시 설정합니다.
- `maxmemory`는 서버 메모리의 60~80% 범위에서 시작합니다.
- 캐시 용도라면 `maxmemory-policy allkeys-lru` 또는 서비스 특성에 맞는 eviction 정책을 지정합니다.
- AOF를 켜면 디스크 IOPS와 fsync 지연을 함께 관측합니다.
- Sentinel/Cluster를 구성할 경우 노드별 장애 테스트를 먼저 수행합니다.
