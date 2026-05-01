# 보안 검토 체크리스트

Redis 설정 및 배포 시 아래 보안 항목을 반드시 확인합니다.

---

## 인증 및 인가

- [ ] `requirepass` 또는 `AUTH` 설정 — 비밀번호 없는 Redis 운영 금지
- [ ] ACL 사용자 설정 — 서비스별 최소 권한 계정 사용
- [ ] 기본 `default` 사용자 비밀번호 설정 또는 비활성화
- [ ] Kubernetes Secret으로 비밀번호 관리 (ConfigMap 사용 금지)

```redis-cli
# 현재 ACL 사용자 목록 확인
ACL LIST

# 기본 사용자 비활성화 (권장)
ACL SETUSER default off
```

---

## 네트워크 접근 제어

- [ ] `bind` 설정으로 리스닝 IP 제한 (0.0.0.0 금지)
- [ ] Redis 포트(6379) 외부 노출 금지 — ClusterIP 서비스 사용
- [ ] Kubernetes NetworkPolicy로 접근 허용 Pod 제한
- [ ] Redis Cluster 내부 포트(16379) 외부 노출 금지

```yaml
# NetworkPolicy 예시 — my-service만 Redis 접근 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-access
  namespace: redis
spec:
  podSelector:
    matchLabels:
      app: redis
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: my-app
          podSelector:
            matchLabels:
              app: my-service
      ports:
        - port: 6379
```

---

## 위험 명령어 비활성화

- [ ] `FLUSHALL`, `FLUSHDB` — 운영 환경 제한 또는 rename
- [ ] `CONFIG` — 외부 접근 시 비활성화
- [ ] `DEBUG` — 반드시 비활성화
- [ ] `KEYS *` — 대규모 키 조회 금지 → `SCAN` 사용

```
# redis.conf — 위험 명령어 rename (비활성화)
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command CONFIG ""
```

---

## TLS 암호화

- [ ] 운영 환경 TLS 활성화 (`tls-port 6380`)
- [ ] 클라이언트-서버 간 평문 전송 금지
- [ ] 인증서 유효 기간 모니터링 (만료 30일 전 알람)
- [ ] TLS 1.2 이상만 허용

---

## 영속성 및 백업

- [ ] RDB 또는 AOF 중 하나 이상 활성화
- [ ] 백업 파일 암호화 저장
- [ ] 백업 주기 및 보존 기간 정책 수립
- [ ] 복구 절차 정기 테스트

---

## 운영 보안

- [ ] Redis 버전 주기적 업데이트 (CVE 확인)
- [ ] Bitnami Helm Chart 최신 버전 유지
- [ ] `protected-mode yes` 설정 (기본값 — 변경 금지)
- [ ] 로그에 민감 정보(키 값, 비밀번호) 노출 금지

---

## Kubernetes 보안

- [ ] Redis Pod SecurityContext 설정 (non-root 실행)
- [ ] PVC 암호화 (EKS: gp3 암호화 StorageClass)
- [ ] RBAC으로 Redis Secret 접근 최소화

```yaml
# Pod SecurityContext
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
```
