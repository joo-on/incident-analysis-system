# Test Data

이 디렉토리는 4개 실패 시나리오 검증을 위한 공통 테스트 데이터를 포함합니다.

## 목적

- 모든 AI 구현이 동일한 테스트 데이터를 사용하여 공정한 비교 보장
- 시나리오 시뮬레이션 시 참조 데이터로 활용
- AI 분석 프롬프트에 포함할 컨텍스트 데이터 제공

## 데이터 구조

```
test-data/
├── scenario-1-db-pool/              # Database Connection Pool Exhaustion
│   ├── connection-metrics.json      # 30분간 풀 사용률 증가 메트릭
│   ├── application-logs.json        # HikariCP 에러 로그
│   └── slow-queries.log             # MySQL 느린 쿼리 로그
│
├── scenario-2-kafka-lag/            # Kafka Consumer Lag Spike
│   ├── consumer-lag-metrics.json    # Lag 증가 추이 (125 → 15,234 messages)
│   ├── kafka-messages.json          # Kafka 메시지 샘플
│   └── consumer-logs.json           # Consumer 처리 지연 로그
│
├── scenario-3-memory-leak/          # Memory Leak Detection
│   ├── jvm-memory-metrics.json      # JVM Heap 사용률 증가 (30% → 98%)
│   ├── gc-logs.json                 # GC 로그 (Full GC 포함)
│   └── heap-dump-summary.txt        # Heap Dump 분석 결과
│
└── scenario-4-cascading/            # Cascading Service Failure
    ├── service-health-metrics.json  # 여러 서비스 헬스 메트릭
    ├── error-logs.json              # 연쇄 에러 로그
    └── trace-data.json              # 분산 추적 데이터 (Zipkin 형식)
```

## 사용 방법

### 1. AI 분석 프롬프트에 포함

```java
String prompt = buildPrompt(incident);

// 컨텍스트 데이터 추가
String metricsContext = loadTestData("scenario-1-db-pool/connection-metrics.json");
String logsContext = loadTestData("scenario-1-db-pool/application-logs.json");

prompt += "\n[메트릭 데이터]\n" + metricsContext;
prompt += "\n[로그 데이터]\n" + logsContext;
```

### 2. 시뮬레이션 검증

```bash
# 시나리오 트리거 후 메트릭 비교
curl -X POST /api/v1/scenarios/scenario-1/trigger

# 실제 생성된 메트릭과 참조 데이터 비교
diff <(curl /api/v1/metrics?service=payment-service&metric=db.pool) \
     reference/test-data/scenario-1-db-pool/connection-metrics.json
```

### 3. 통합 테스트

```java
@Test
public void testScenario1WithReferenceData() {
    // 참조 데이터 로드
    List<MetricEvent> expectedMetrics = loadJson(
        "test-data/scenario-1-db-pool/connection-metrics.json",
        MetricEvent.class
    );

    // 시나리오 실행
    triggerScenario("scenario-1");

    // 실제 메트릭과 비교
    List<MetricEvent> actualMetrics = getActualMetrics();
    assertThat(actualMetrics).hasSizeGreaterThanOrEqualTo(expectedMetrics.size());
}
```

## 데이터 상세

### Scenario 1: Database Connection Pool Exhaustion

**connection-metrics.json**
- 시간 범위: 14:00 ~ 14:30 (30분)
- 데이터 포인트: 8개 (5분 간격)
- 풀 사용률: 32.5% → 100%
- Pending 요청: 0 → 184개

**application-logs.json**
- 로그 레벨: WARN, ERROR
- 주요 에러: `Connection is not available`, `Thread starvation`
- 스택 트레이스 포함
- 5개 로그 엔트리

**slow-queries.log**
- MySQL Slow Query Log 형식
- 쿼리 실행 시간: 8초 ~ 20초
- Lock 시간: 0초 ~ 5초
- 4개 느린 쿼리

### Scenario 2: Kafka Consumer Lag Spike

**consumer-lag-metrics.json**
- 시간 범위: 14:00 ~ 14:30 (30분)
- Lag 증가: 125 → 15,234 messages
- Partition: 0 (metrics topic)
- Consumer Group: incident-detection-group

**kafka-messages.json**
- 5개 샘플 메시지
- Topic: metrics, logs
- 메시지 구조: timestamp, service, metric, value

**consumer-logs.json**
- Consumer 처리 지연 로그
- 평균 처리 시간: 487ms (임계값: 100ms 초과)
- Throughput gap: 898 msg/sec

### Scenario 3: Memory Leak Detection

**jvm-memory-metrics.json**
- 시간 범위: 14:00 ~ 14:36 (36분)
- Heap 사용률: 30% → 98%
- 선형 증가 패턴 (메모리 누수 징후)
- 9개 데이터 포인트

**gc-logs.json**
- Young GC: 5회 (평균 142ms)
- Mixed GC: 1회 (1,234ms)
- Full GC: 1회 (8,234ms) - 메모리 거의 회수 못함
- GC overhead limit exceeded 경고

**heap-dump-summary.txt**
- Eclipse MAT 분석 결과 형식
- Top 메모리 소비자: HashMap (60%), ArrayList (25%)
- Leak Suspects: 3개 (confidence 95%, 87%, 72%)
- 코드 위치 및 권장사항 포함

### Scenario 4: Cascading Service Failure

**service-health-metrics.json**
- 6개 서비스 헬스 메트릭
- 장애 전파 순서:
  1. payment-gateway (DOWN)
  2. payment-service (Thread pool 100%)
  3. order-service (Timeout 51.7%)
  4. api-gateway (Error rate 87.3%)

**error-logs.json**
- 7개 연쇄 에러 로그
- Connection refused → Timeout → Thread pool exhaustion
- Circuit breaker 상태: CLOSED (문제!)
- Retry 시도 기록

**trace-data.json**
- Distributed tracing (Zipkin/Jaeger 형식)
- 4개 span (api-gateway → order → payment → gateway)
- 총 duration: 30,567ms
- Failure chain 요약 포함

## 데이터 무결성

### 검증 체크리스트

- [ ] 모든 JSON 파일이 valid JSON 형식
- [ ] 타임스탬프가 시간 순서대로 정렬
- [ ] 메트릭 값이 논리적으로 타당 (예: 사용률 0~100%)
- [ ] 로그 레벨이 적절 (ERROR > WARN > INFO)
- [ ] 스택 트레이스가 실제 Java 형식

### 검증 스크립트

```bash
# JSON 파일 유효성 검사
find . -name "*.json" -exec sh -c 'jq empty {} || echo "Invalid: {}"' \;

# 타임스탬프 순서 확인
jq -r '.[].timestamp' scenario-1-db-pool/connection-metrics.json | \
  sort -c || echo "Timestamps not sorted"
```

## 버전 관리

- **Version**: 1.0.0
- **Last Updated**: 2026-02-14
- **Author**: AI Benchmark Project Team

### 변경 이력

- **v1.0.0** (2026-02-14): 초기 테스트 데이터 생성
  - 4개 시나리오별 3-4개 파일
  - 총 13개 파일
  - 실제 프로덕션 장애 패턴 기반

## 주의사항

1. **데이터 수정 금지**: 공정한 비교를 위해 한번 생성된 테스트 데이터는 수정하지 않습니다.
2. **추가 데이터**: 새로운 시나리오 추가 시 별도 디렉토리 생성
3. **버전 관리**: 테스트 데이터 변경 시 버전 업데이트 및 변경 이력 기록

## 라이선스

이 테스트 데이터는 프로젝트의 일부로 MIT 라이선스 하에 제공됩니다.
