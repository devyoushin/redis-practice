# /new-doc

신규 Redis 지식 문서를 스캐폴딩합니다.

## 사용법

```
/new-doc <카테고리> <주제>
```

**카테고리**: `data-structures`, `operations`, `performance`, `security`, `observability`

## 예시

```
/new-doc performance pipeline-optimization
/new-doc operations backup-restore
/new-doc data-structures pub-sub
```

## 실행 내용

`$ARGUMENTS`를 파싱하여 `docs/guides/<카테고리>/<주제>.md` 파일을 `docs/templates/service-doc.md` 형식으로 생성합니다.

다음 규칙을 따릅니다:
1. `docs/rules/doc-writing.md` — 5개 섹션(개요/설명/트러블슈팅/모니터링/TIP) 필수
2. `docs/rules/redis-conventions.md` — redis-cli 명령어 형식 준수
3. `docs/rules/monitoring.md` — Prometheus 지표 테이블 포함
4. 한국어 작성, 영어 원문 병기 (예: 메모리(Memory))
5. 섹션 4(모니터링)에 `redis-cli INFO` 또는 Prometheus 지표 반드시 포함
