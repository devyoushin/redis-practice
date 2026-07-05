# Redis Helm/Kubernetes 설치

Redis는 Kubernetes에 배포할 수 있지만, 항상 좋은 선택은 아닙니다. Redis는 상태 저장 데이터베이스에 가깝게 운영되는 경우가 많고, Pod 재스케줄링, PVC 성능, 네트워크 지연, failover 타이밍이 서비스 안정성에 직접 영향을 줍니다.

## Kubernetes 배포를 피하는 편이 나은 경우

- Redis가 세션, 랭킹, 큐 등 서비스 핵심 상태를 가진다.
- 스토리지 클래스의 IOPS, latency, attach/detach 시간을 예측하기 어렵다.
- Redis 장애 조치, 백업/복구, resharding을 정기적으로 테스트하지 않는다.
- 노드 drain, cluster autoscaler, zone 장애 상황에서 Redis 동작을 검증하지 않았다.
- 단순히 “Kubernetes에 다 올리고 싶다”는 이유뿐이다.

## Kubernetes 배포를 검토할 수 있는 경우

- 플랫폼 표준이 Kubernetes이고 StatefulSet 운영 경험이 충분하다.
- Redis 데이터 손실 허용 범위와 복구 절차가 명확하다.
- Pod anti-affinity, topology spread, PDB, PVC 백업 정책이 준비되어 있다.
- Sentinel 또는 Cluster failover를 실제 장애 시나리오로 검증했다.

## Helm 예시

이 레포의 Helm values 예시는 `ops/config/`에 있습니다.

| 파일 | 목적 |
|------|------|
| `ops/config/redis-sentinel-values.yaml` | Sentinel 기반 HA 구성 |
| `ops/config/redis-cluster-values.yaml` | Redis Cluster 샤딩 구성 |

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install redis bitnami/redis \
  -n redis \
  --create-namespace \
  -f ops/config/redis-sentinel-values.yaml \
  --set auth.password="$(openssl rand -base64 32)"
```

## 필수 점검

- `resources.requests.memory`와 `maxmemory`를 같이 맞춥니다.
- `maxmemory`는 컨테이너 메모리 limit보다 낮게 잡아 OOMKilled를 피합니다.
- `PodDisruptionBudget`으로 자발적 중단을 제한합니다.
- `anti-affinity` 또는 topology spread로 Redis Pod를 노드/존에 분산합니다.
- PVC 스냅샷, AOF/RDB 백업, 복구 리허설을 준비합니다.
- rolling upgrade 전 failover와 client reconnect 동작을 검증합니다.

