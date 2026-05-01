# /review-doc

Redis 지식 문서의 품질을 검토합니다.

## 사용법

```
/review-doc <파일경로>
```

## 예시

```
/review-doc docs/performance/memory-management.md
/review-doc docs/operations/redis-cluster.md
```

## 검토 항목

`$ARGUMENTS`로 지정된 파일을 읽고 다음 기준으로 검토합니다:

### 구조 검토
- [ ] 5개 섹션(1.개요 / 2.설명 / 3.트러블슈팅 / 4.모니터링 / 5.TIP) 모두 포함
- [ ] 섹션 2에 핵심 개념 표 또는 다이어그램 포함
- [ ] 섹션 3에 증상/원인/해결 형식 트러블슈팅 최소 2개

### 코드 품질
- [ ] redis-cli 명령어 즉시 실행 가능 (플레이스홀더 명확히 표시)
- [ ] 설정 값에 단위 주석 포함 (예: `maxmemory 4gb  # 4GB`)
- [ ] Bitnami Helm values 형식 준수 (운영 관련 문서)

### 내용 품질
- [ ] 한국어 본문 + 영어 원문 병기
- [ ] 섹션 4에 Prometheus 지표 테이블 또는 `redis-cli INFO` 명령어 포함
- [ ] 운영 환경 권장값과 기본값 구분

검토 결과를 항목별로 Pass/Fail로 출력하고, Fail 항목에 대한 개선 제안을 제시합니다.
