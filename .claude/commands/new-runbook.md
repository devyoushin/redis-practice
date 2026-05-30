# /new-runbook

Redis 운영 Runbook을 생성합니다.

## 사용법

```
/new-runbook <카테고리> <주제>
```

## 예시

```
/new-runbook operations failover-recovery
/new-runbook operations memory-eviction-response
/new-runbook operations cluster-node-replacement
```

## 실행 내용

`$ARGUMENTS`를 파싱하여 `docs/<카테고리>/<주제>-runbook.md` 파일을 `docs/templates/runbook.md` 형식으로 생성합니다.

Runbook 필수 포함 항목:
1. **사전 조건**: 실행 전 확인 사항, 필요 권한
2. **영향 범위**: 영향받는 서비스, 다운타임 여부
3. **단계별 절차**: 번호 매긴 순차 실행 명령어 (redis-cli 복붙 가능)
4. **롤백 방법**: 실패 시 이전 상태 복구 절차
5. **완료 확인**: 정상 완료 판단 기준
