# Reference

모든 구현에서 공통으로 사용할 참고 자료 및 테스트 데이터를 저장합니다.

## 디렉토리 구성

### `test-data/`
실패 시나리오 검증을 위한 테스트 데이터:
- 샘플 로그 파일
- 메트릭 데이터 (CPU, Memory, Network)
- Kafka 메시지 샘플
- 데이터베이스 덤프 (시나리오별)

### `baseline-implementation/` (선택)
수동으로 작성한 기준 구현:
- AI 구현과 비교하기 위한 "정답" 코드
- 성능 및 품질 벤치마크 기준선

### `libraries-research.md`
프로젝트에서 사용 가능한 라이브러리 조사:
- Spring Boot Starter 목록
- Kafka Client 라이브러리
- AI API 클라이언트 (Claude, OpenAI)
- 로깅, 모니터링 도구

## 원칙

- ✅ 모든 구현은 동일한 테스트 데이터를 사용해야 합니다
- ✅ 공정한 비교를 위해 테스트 조건을 통일합니다
- ✅ 테스트 데이터는 버전 관리되며, 변경 시 문서화합니다

## 테스트 데이터 생성 가이드

```bash
# 예시: 시나리오별 테스트 데이터 생성
test-data/
├── scenario-1-db-pool/
│   ├── application-logs.json
│   ├── connection-metrics.csv
│   └── database-slow-queries.log
├── scenario-2-kafka-lag/
│   ├── consumer-lag-metrics.json
│   └── kafka-messages.json
└── ...
```
