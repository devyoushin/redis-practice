# Redis 설치 방식 선택

Redis 설치는 목적에 따라 다르게 선택합니다. 운영 기준으로는 VM 또는 Bare Metal에서 `systemd`로 관리하는 방식이 가장 예측 가능하고, 개발/검증 환경은 Docker Compose가 편합니다. Kubernetes 배포는 가능하지만 Redis의 상태 저장, 장애 복구, 네트워크 지연, 스토리지 성능을 충분히 통제할 수 있을 때만 선택합니다.

## 빠른 선택 기준

| 목적 | 권장 방식 | 이유 |
|------|-----------|------|
| 운영 단일 노드, Sentinel, Cluster | systemd | OS, 디스크, 커널 파라미터, 장애 복구를 직접 통제하기 좋음 |
| 로컬 개발, 기능 검증, 간단한 PoC | Docker Compose | 빠르게 올리고 내리기 쉬움 |
| 패키지 기반 표준 배포 | RPM/DEB | OS 패키지 관리와 보안 업데이트 흐름을 사용 가능 |
| 특정 버전 고정, 소스 옵션 확인 | tar/source | Redis 버전과 빌드 옵션을 직접 통제 가능 |
| Kubernetes 환경 통합 | Helm | 플랫폼 표준이 Kubernetes이고 StatefulSet 운영 역량이 있을 때만 |

## 문서 순서

1. `systemd.md` - 운영 서버에 Redis를 서비스로 등록하는 기본 절차
2. `docker-compose.md` - 로컬/검증 환경에서 Docker Compose로 실행
3. `package-tar.md` - RPM/DEB 패키지와 tar/source 설치 방식
4. `helm-kubernetes.md` - Kubernetes/Helm 배포 판단 기준과 주의점

## 운영 권장 결론

- Redis를 핵심 캐시/세션/랭킹 저장소로 쓴다면 `systemd` 기반 VM 배포를 우선 검토합니다.
- 개발자가 빠르게 테스트해야 하는 환경은 Docker Compose를 사용합니다.
- Kubernetes에 올릴 수는 있지만, Redis가 상태 저장 시스템이라는 점 때문에 무조건 좋은 선택은 아닙니다.
- Kubernetes에서 Redis를 운영하려면 PVC 성능, PodDisruptionBudget, anti-affinity, 백업/복구, failover 테스트가 선행되어야 합니다.

