# Redis RPM/DEB/tar 설치

Redis는 OS 패키지로 설치하거나, 특정 버전을 tar/source로 받아 빌드할 수 있습니다. 운영 표준화가 목적이면 패키지 설치가 편하고, 특정 Redis 버전 고정이나 빌드 옵션 확인이 목적이면 tar/source 방식이 적합합니다.

## RPM/DEB 패키지 설치

배포판 저장소에서 제공하는 Redis는 설치와 업데이트가 단순합니다. 단, 배포판에 따라 Redis 버전이 최신보다 낮을 수 있으므로 운영 요구 버전과 먼저 비교합니다.

### RHEL 계열

```bash
sudo dnf install -y redis
sudo systemctl enable --now redis
redis-cli ping
```

### Debian/Ubuntu 계열

```bash
sudo apt-get update
sudo apt-get install -y redis-server
sudo systemctl enable --now redis-server
redis-cli ping
```

## tar/source 설치

특정 Redis 버전을 직접 고정해야 할 때 사용합니다.

```bash
curl -LO https://download.redis.io/releases/redis-7.2.5.tar.gz
tar xzf redis-7.2.5.tar.gz
cd redis-7.2.5
make
make test
sudo make install
```

설치 후에는 `systemd.md`의 사용자, 디렉터리, unit 등록 절차를 적용합니다.

## 선택 기준

| 방식 | 장점 | 주의점 |
|------|------|--------|
| RPM/DEB | OS 패키지 관리, 업데이트 용이 | 배포판 제공 버전이 낮을 수 있음 |
| tar/source | 버전 고정, 빌드 옵션 통제 | 패치/업그레이드 자동화가 필요 |

## 운영 메모

- 설치 방식과 무관하게 운영 설정은 `/etc/redis/redis.conf`로 표준화합니다.
- Redis 바이너리 버전, 설정 파일, systemd unit, 백업 절차를 함께 기록합니다.
- source 설치는 업그레이드 롤백 경로를 미리 준비합니다.

