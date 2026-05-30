# /add-troubleshooting

기존 Redis 문서에 트러블슈팅 항목을 추가합니다.

## 사용법

```
/add-troubleshooting <파일경로> <증상설명>
```

## 예시

```
/add-troubleshooting docs/guides/performance/memory-management.md "OOM killer가 Redis 프로세스를 종료시킴"
/add-troubleshooting docs/guides/operations/redis-cluster.md "CLUSTERDOWN 오류로 쓰기 실패"
```

## 실행 내용

`$ARGUMENTS`를 파싱하여:

1. 지정 파일의 `## 3. 트러블슈팅` 섹션을 찾음
2. 다음 형식으로 새 항목 추가:

```markdown
### 3.N <증상 요약>

#### 증상
- <구체적 증상>
- <관련 로그 또는 오류 메시지>

#### 원인
- <근본 원인>

#### 해결 방법
```redis-cli
# 진단 명령어
# 해결 명령어
```

> **예방책**: <재발 방지 방법>
```

3. 기존 트러블슈팅 번호 순서 유지
4. 새 항목이 기존 내용과 중복되지 않는지 확인
