# /search-kb

Redis 지식 베이스에서 키워드로 관련 문서를 검색합니다.

## 사용법

```
/search-kb <키워드>
```

## 예시

```
/search-kb eviction memory
/search-kb cluster failover
/search-kb slow log latency
/search-kb tls authentication
```

## 실행 내용

`$ARGUMENTS` 키워드를 사용하여:

1. `docs/` 디렉토리 전체 마크다운 파일에서 키워드 검색
2. 관련 문서 목록과 매칭된 섹션 요약 반환
3. 관련도 높은 순으로 정렬하여 출력

출력 형식:
```
## 검색 결과: "<키워드>"

### 1. docs/guides/performance/memory-management.md
> 섹션 2.1 핵심 개념 — Eviction Policy 설명...

### 2. docs/guides/operations/redis-cluster.md
> 섹션 3.2 트러블슈팅 — Cluster Failover 절차...
```
