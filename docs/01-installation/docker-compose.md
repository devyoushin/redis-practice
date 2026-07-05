# Redis Docker Compose 설치

Docker Compose는 로컬 개발, 기능 검증, 간단한 PoC에 적합합니다. 운영 핵심 Redis를 장기간 운영하는 방식으로는 systemd 또는 전용 관리형 서비스를 우선 검토합니다.

## 실행

```bash
cd ops/install
docker compose up -d
```

## 확인

```bash
docker compose ps
docker compose exec redis redis-cli -a redis-local-password ping
docker compose exec redis redis-cli -a redis-local-password INFO memory
```

## 중지

```bash
docker compose down
```

데이터까지 삭제하려면 볼륨을 함께 제거합니다.

```bash
docker compose down -v
```

## 주의점

- Compose 예제의 비밀번호는 로컬 검증용입니다. 운영 값으로 재사용하지 않습니다.
- bind mount 또는 named volume을 사용해 `/data`를 유지합니다.
- Linux 운영 서버에서 장기 운영할 경우 Docker 데몬 장애, 로그 관리, 재시작 정책까지 함께 설계해야 합니다.
- Redis 성능 검증은 컨테이너 내부 수치만 보지 말고 호스트 CPU, 메모리, 디스크 I/O를 함께 확인합니다.

