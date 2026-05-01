# TLS 암호화

Redis TLS(Transport Layer Security) 암호화는 클라이언트와 서버 간 통신 데이터를 암호화합니다. Redis 6.0+에서 네이티브 TLS를 지원하며, Bitnami Helm Chart에서 인증서 설정이 가능합니다.

---

## 1. 개요

Redis TLS 구성:
- **TLS 포트**: 6380 (기본 6379 대신 또는 병행 사용)
- **인증서**: CA 인증서, 서버 인증서/키
- **mTLS**: 클라이언트 인증서도 검증 (선택)
- **TLS 버전**: TLS 1.2 이상 권장

---

## 2. 설명

### 2.1 핵심 개념

#### TLS 설정 옵션

| 설정 | 설명 |
|------|------|
| `tls-port 6380` | TLS 포트 (6379와 병행 또는 단독) |
| `port 0` | 평문 포트 비활성화 (TLS 전용) |
| `tls-cert-file` | 서버 인증서 경로 |
| `tls-key-file` | 서버 개인키 경로 |
| `tls-ca-cert-file` | CA 인증서 경로 |
| `tls-auth-clients yes` | 클라이언트 인증서 검증 (mTLS) |
| `tls-replication yes` | 복제 연결 TLS 적용 |
| `tls-cluster yes` | Cluster 내부 통신 TLS 적용 |

#### 인증서 종류

```
Self-signed (자체 서명):
  → 개발/내부 환경
  → 클라이언트 Truststore에 CA 인증서 추가 필요

CA-signed (CA 서명):
  → 운영 환경 권장
  → Let's Encrypt, AWS ACM, 기업 CA

cert-manager (Kubernetes):
  → 자동 인증서 발급/갱신
  → Bitnami Helm과 연동 가능
```

### 2.2 실무 적용 코드

#### 자체 서명 인증서 생성

```bash
# CA 키 및 인증서 생성
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=Redis CA"

# 서버 키 및 CSR 생성
openssl genrsa -out redis.key 2048
openssl req -new -key redis.key -out redis.csr \
  -subj "/CN=redis-master.redis.svc.cluster.local"

# CA로 서버 인증서 서명
openssl x509 -req -days 365 \
  -in redis.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out redis.crt

# 인증서 확인
openssl x509 -in redis.crt -text -noout | grep -E "Subject|Issuer|Not"
```

#### Kubernetes Secret으로 인증서 관리

```bash
# TLS Secret 생성
kubectl create secret generic redis-tls \
  -n redis \
  --from-file=ca.crt=./ca.crt \
  --from-file=redis.crt=./redis.crt \
  --from-file=redis.key=./redis.key
```

#### redis.conf TLS 설정

```
# 평문 포트 비활성화 + TLS 전용
port 0
tls-port 6380

# 인증서 설정
tls-cert-file /tls/redis.crt
tls-key-file /tls/redis.key
tls-ca-cert-file /tls/ca.crt

# mTLS (클라이언트 인증서 검증)
tls-auth-clients yes              # yes: 필수, optional: 선택, no: 비활성

# 복제 연결 TLS
tls-replication yes

# TLS 버전 제한
tls-protocols "TLSv1.2 TLSv1.3"

# 암호화 스위트 (강력한 암호화만 허용)
tls-ciphers "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256"
tls-ciphersuites "TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256"
tls-prefer-server-ciphers yes
```

#### Bitnami Redis Helm TLS 설정

```yaml
# redis-sentinel-values.yaml
tls:
  enabled: true
  existingSecret: redis-tls          # 위에서 생성한 Secret
  certFilename: redis.crt
  certKeyFilename: redis.key
  certCAFilename: ca.crt
  authClients: true                  # mTLS 활성화

# TLS 포트 설정
master:
  containerPorts:
    redis: 6380
```

#### TLS 연결 테스트

```bash
# redis-cli TLS 연결
redis-cli -h redis-master.redis.svc.cluster.local \
  -p 6380 \
  -a <REDIS_PASSWORD> \
  --tls \
  --cacert /tls/ca.crt \
  PING

# mTLS 연결 (클라이언트 인증서 포함)
redis-cli -h redis-master.redis.svc.cluster.local \
  -p 6380 \
  -a <REDIS_PASSWORD> \
  --tls \
  --cacert /tls/ca.crt \
  --cert /tls/client.crt \
  --key /tls/client.key \
  PING
```

#### Spring Boot TLS 클라이언트 설정

```yaml
spring:
  data:
    redis:
      host: redis-master.redis.svc.cluster.local
      port: 6380
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: true
      lettuce:
        ssl:
          enabled: true
```

```java
// Lettuce TLS 상세 설정 (Java Config)
@Bean
public LettuceConnectionFactory redisConnectionFactory() {
    SslOptions sslOptions = SslOptions.builder()
        .truststore(new FileResource(new File("/tls/truststore.jks")))
        .truststorePassword("truststorepassword".toCharArray())
        // mTLS 사용 시
        .keystore(new FileResource(new File("/tls/client-keystore.p12")))
        .keystorePassword("keystorepassword".toCharArray())
        .build();

    ClientOptions clientOptions = ClientOptions.builder()
        .sslOptions(sslOptions)
        .build();

    LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
        .useSsl()
        .clientOptions(clientOptions)
        .commandTimeout(Duration.ofMillis(2000))
        .build();

    RedisStandaloneConfiguration serverConfig = new RedisStandaloneConfiguration();
    serverConfig.setHostName("redis-master.redis.svc.cluster.local");
    serverConfig.setPort(6380);
    serverConfig.setPassword(RedisPassword.of(redisPassword));

    return new LettuceConnectionFactory(serverConfig, clientConfig);
}
```

### 2.3 Best Practice

- 운영 환경에서 `port 0` + `tls-port 6380`으로 평문 포트 비활성화
- 인증서 갱신 주기 모니터링 (만료 30일 전 알람)
- Kubernetes에서는 cert-manager로 자동 인증서 관리 권장
- `tls-auth-clients yes` (mTLS)로 클라이언트 인증 강화

---

## 3. 트러블슈팅

### 3.1 SSL Handshake 실패

#### 증상
```
javax.net.ssl.SSLHandshakeException: PKIX path building failed
```

#### 원인
- 클라이언트 Truststore에 CA 인증서 미포함
- 인증서 만료

#### 해결 방법
```bash
# 인증서 만료 확인
openssl x509 -in /tls/redis.crt -noout -enddate

# CA 인증서를 JKS Truststore에 추가
keytool -import -alias redis-ca -file /tls/ca.crt \
  -keystore truststore.jks -storepass truststorepassword -noprompt

# openssl로 TLS 연결 테스트
openssl s_client -connect redis-master.redis.svc.cluster.local:6380 \
  -CAfile /tls/ca.crt
```

### 3.2 mTLS 클라이언트 인증 실패

#### 증상
```
ERR unencrypted connection is not allowed.
```

#### 원인
- `tls-auth-clients yes` 설정인데 클라이언트 인증서 미제공

#### 해결 방법
```bash
# redis-cli에 클라이언트 인증서 추가
redis-cli --tls --cacert ca.crt --cert client.crt --key client.key \
  -h redis-master.redis.svc.cluster.local -p 6380 -a <PASSWORD> PING
```

### 3.3 인증서 만료로 연결 오류

#### 증상
- 특정 날짜 이후 Redis 연결 오류 급증

#### 원인
- 서버 또는 클라이언트 인증서 만료

#### 해결 방법
```bash
# 인증서 만료 확인
openssl x509 -in redis.crt -noout -enddate

# 인증서 갱신 후 Secret 업데이트
kubectl create secret generic redis-tls \
  -n redis \
  --from-file=ca.crt=./ca-new.crt \
  --from-file=redis.crt=./redis-new.crt \
  --from-file=redis.key=./redis-new.key \
  --dry-run=client -o yaml | kubectl apply -f -

# Redis Pod 재시작 (새 인증서 로드)
kubectl rollout restart statefulset redis-master -n redis
```

---

## 4. 모니터링 및 확인

```bash
# TLS 연결 상태 확인
redis-cli --tls --cacert /tls/ca.crt \
  -h redis-master.redis.svc.cluster.local -p 6380 -a <PASSWORD> PING

# 인증서 만료일 확인
openssl x509 -in /tls/redis.crt -noout -enddate

# TLS 정보 확인 (redis-cli INFO)
redis-cli --tls --cacert /tls/ca.crt \
  -h redis-master.redis.svc.cluster.local -p 6380 -a <PASSWORD> \
  INFO server | grep tls
```

| Prometheus 지표 | 설명 | 알람 기준 |
|----------------|------|---------|
| `redis_tls_enabled` | TLS 활성화 여부 | 운영환경 0이면 경고 |
| `ssl_certificate_expiry_seconds` | 인증서 만료까지 남은 시간 | < 30일 |

---

## 5. TIP

- cert-manager `Certificate` 리소스로 Redis TLS 인증서 자동 갱신 구성 가능
- Redis Sentinel TLS 설정 시 Sentinel 포트(26379)도 TLS 적용 필요
- `tls-protocols "TLSv1.3"` 으로 TLS 1.3만 허용하면 보안 강화 (클라이언트 호환성 확인 필요)
- 참고: [Redis TLS Support](https://redis.io/docs/management/security/encryption/)
