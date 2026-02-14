# Scenario 2: Kafka Consumer Lag Spike

## 개요

**시나리오 ID**: `scenario-2`
**시나리오 명**: Kafka Consumer Lag Spike
**심각도**: WARNING → CRITICAL (지속 시)
**예상 발생 빈도**: 높음 (주 1-2회)

### 설명
Kafka Consumer가 메시지를 생산 속도만큼 빠르게 소비하지 못해 Consumer Lag이 급증하는 상황입니다. 처리 지연이 누적되어 실시간 장애 감지 및 분석에 영향을 줍니다.

### 비즈니스 영향
- 장애 감지 지연 (실시간성 저하)
- 오래된 메트릭/로그 데이터 기반 분석으로 부정확한 결과
- 중요한 CRITICAL 알림 누락 위험
- 시스템 전체 백프레셔(backpressure) 발생 가능

---

## 근본 원인 (Root Cause)

### 주요 원인

1. **Slow Message Processing**
   ```java
   // ❌ 느린 처리 로직
   @KafkaListener(topics = "metrics")
   public void consume(MetricEvent event) {
       // DB에 동기적으로 저장 (느림)
       metricRepository.save(event);

       // 외부 API 호출 (블로킹)
       externalService.call(event);

       // 복잡한 계산
       complexAnalysis(event);

       // 총 처리 시간: ~500ms/message
       // → 초당 2개만 처리 가능
   }
   ```

2. **Insufficient Consumer Instances**
   - Consumer Group 인스턴스 수 < Partition 수
   - 스케일링 부족

3. **Sudden Traffic Spike**
   - 예상치 못한 트래픽 급증
   - 대규모 배치 작업으로 메시지 대량 발생

4. **Consumer Pause/Restart**
   - Consumer 장애 또는 배포로 일시 중단
   - Rebalancing으로 인한 일시적 중단

5. **Downstream Service Bottleneck**
   - MySQL slow query로 메시지 처리 지연
   - Redis 응답 지연

---

## 시뮬레이션 방법

### API 트리거

```bash
curl -X POST http://localhost:8080/api/v1/scenarios/scenario-2/trigger \
  -H "Content-Type: application/json" \
  -d '{
    "duration": "PT5M",
    "intensity": "HIGH"
  }'
```

### 시뮬레이션 로직

```java
@Service
public class KafkaLagScenario {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private KafkaListenerEndpointRegistry registry;

    public void execute(String intensity) {
        // 1. Consumer를 일시 중지하여 lag 발생 유도
        MessageListenerContainer container = registry
            .getListenerContainer("incident-detection-group");
        container.pause();

        // 2. 대량의 메시지 발행 (생산 속도 증가)
        int messageCount = intensity.equals("HIGH") ? 50000 : 20000;
        int messagesPerSecond = 1000;

        ExecutorService executor = Executors.newSingleThreadExecutor();
        executor.submit(() -> {
            for (int i = 0; i < messageCount; i++) {
                MetricEvent event = createMetricEvent();
                kafkaTemplate.send("metrics", event.toJson());

                if (i % messagesPerSecond == 0) {
                    try {
                        Thread.sleep(1000); // 초당 1000개 발행
                    } catch (InterruptedException e) {}
                }
            }
        });

        // 3. 30초 후 Consumer 재개 (느린 처리로 lag 증가)
        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                container.resume();

                // 처리 속도 의도적으로 느리게
                container.getContainerProperties().setPollTimeout(5000);
            }
        }, 30000);

        // 4. 5분 후 정상화
        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                container.getContainerProperties().setPollTimeout(3000);
            }
        }, 300000);
    }
}
```

### 생성되는 메트릭 데이터

```json
{
  "timestamp": "2026-02-14T14:30:12.345Z",
  "service": "incident-detection-engine",
  "host": "detection-prod-01",
  "metric": "kafka.consumer.lag",
  "value": 15234,
  "unit": "messages",
  "metadata": {
    "topic": "metrics",
    "partition": 0,
    "consumerGroup": "incident-detection-group",
    "currentOffset": 12345,
    "logEndOffset": 27579,
    "lag": 15234
  }
}
```

### 생성되는 로그 데이터

```json
{
  "timestamp": "2026-02-14T14:30:12.345Z",
  "level": "WARN",
  "service": "incident-detection-engine",
  "host": "detection-prod-01",
  "logger": "org.springframework.kafka.listener.KafkaMessageListenerContainer",
  "message": "Consumer lag detected: 15234 messages behind on partition metrics-0",
  "context": {
    "consumerGroup": "incident-detection-group",
    "topic": "metrics",
    "partition": 0,
    "lag": 15234
  }
}
```

---

## 감지 기준 (Detection Rule)

### 규칙 정의

```java
@Component
public class KafkaConsumerLagRule implements DetectionRule {

    private static final int CRITICAL_LAG_THRESHOLD = 10000;
    private static final int WARNING_LAG_THRESHOLD = 5000;

    @Override
    public boolean evaluate(MetricEvent event) {
        return event.getMetricName().equals("kafka.consumer.lag")
            && event.getValue() > CRITICAL_LAG_THRESHOLD;
    }

    @Override
    public Severity getSeverity() {
        // Lag 크기에 따라 동적으로 심각도 결정
        double lag = event.getValue();
        if (lag > CRITICAL_LAG_THRESHOLD) {
            return Severity.CRITICAL;
        } else if (lag > WARNING_LAG_THRESHOLD) {
            return Severity.WARNING;
        }
        return Severity.INFO;
    }

    @Override
    public String getDescription() {
        return "Kafka consumer lag > 10,000 messages";
    }
}
```

### 임계값

| 메트릭 | 임계값 | 지속 시간 | 심각도 |
|--------|--------|-----------|--------|
| kafka.consumer.lag | > 10,000 msgs | 즉시 | CRITICAL |
| kafka.consumer.lag | > 5,000 msgs | 1분 | WARNING |
| kafka.consumer.lag | > 1,000 msgs | 5분 | INFO |

### 예상 감지 시간

- **목표**: 60초 이내
- **프로세스**:
  1. Lag 메트릭 수집 (30초 주기)
  2. Rule Engine 평가 - 1초
  3. Incident 생성 - 1초
  4. **총 소요 시간**: ~32초

---

## AI 분석 예상 결과

### Root Cause Analysis

```json
{
  "root_cause": {
    "summary": "Slow database writes in MetricEventProcessor causing message processing backlog",
    "details": "The Kafka consumer is processing messages at ~100 msg/sec, but the producer rate is ~1000 msg/sec. Analysis of consumer logs shows that each message takes ~500ms to process due to synchronous database writes and lack of batching. This 10x throughput mismatch causes lag accumulation.",
    "confidence": 0.90,
    "evidence": [
      "Consumer lag increased from 100 to 15,234 messages in 15 minutes",
      "Consumer logs show average processing time of 500ms per message",
      "Database query logs show individual INSERT statements instead of batch inserts",
      "No consumer rebalancing or restart events detected"
    ]
  }
}
```

### Impact Assessment

```json
{
  "impact": {
    "affected_services": ["incident-detection-engine"],
    "severity": "CRITICAL",
    "business_impact": "Real-time incident detection delayed by ~2.5 hours at current rate. Critical alerts may be missed.",
    "downstream_effects": [
      "Incident detection latency increased from <1s to ~150s",
      "AI analysis triggered on stale data (15 minutes old)",
      "Slack alerts delayed",
      "Dashboard metrics outdated"
    ],
    "risk": "If lag continues to grow, memory exhaustion possible in Kafka broker"
  }
}
```

### Recommended Actions

```json
{
  "recommended_actions": [
    {
      "priority": 1,
      "action": "Scale out consumer instances from 3 to 6 (match partition count)",
      "expected_result": "Double throughput to ~200 msg/sec, reduce lag accumulation rate",
      "estimated_duration": "PT5M",
      "risk": "LOW"
    },
    {
      "priority": 2,
      "action": "Implement batch processing: accumulate 100 messages before DB write",
      "expected_result": "Reduce DB round-trips from 1000/sec to 10/sec, increase throughput to ~800 msg/sec",
      "estimated_duration": "PT2H",
      "risk": "MEDIUM - Potential message loss if consumer crashes before commit"
    },
    {
      "priority": 3,
      "action": "Enable asynchronous processing: send metrics to Redis queue, separate worker for DB writes",
      "expected_result": "Decouple Kafka consumption from DB writes, throughput > 2000 msg/sec",
      "estimated_duration": "PT4H",
      "risk": "MEDIUM - Complexity increase, need monitoring for queue backlog"
    },
    {
      "priority": 4,
      "action": "Add index on 'detected_at' column in incidents table to speed up writes",
      "expected_result": "Reduce DB write latency by ~30%",
      "estimated_duration": "PT30M",
      "risk": "LOW"
    }
  ]
}
```

### Timeline

```json
{
  "timeline": [
    {
      "time": "2026-02-14T14:00:00.000Z",
      "event": "Normal operation, lag < 100 messages"
    },
    {
      "time": "2026-02-14T14:15:00.000Z",
      "event": "Traffic spike detected: producer rate increased from 100/s to 1000/s"
    },
    {
      "time": "2026-02-14T14:20:00.000Z",
      "event": "Consumer lag started accumulating: 500 messages"
    },
    {
      "time": "2026-02-14T14:25:00.000Z",
      "event": "Lag reached 5,000 messages (WARNING threshold)"
    },
    {
      "time": "2026-02-14T14:30:00.000Z",
      "event": "Lag exceeded 10,000 messages (CRITICAL threshold), incident triggered"
    }
  ]
}
```

### Prevention

```
1. Auto-scaling: Configure Kubernetes HPA based on Kafka lag metrics
2. Batch Processing: Always use batch inserts for high-throughput scenarios
3. Monitoring: Set up alerts at 1K, 5K, 10K lag thresholds with different severities
4. Load Testing: Regular load tests to identify throughput bottlenecks
5. Circuit Breaker: Implement backpressure mechanism to slow down producers if needed
6. Partitioning Strategy: Ensure partition count >= max expected consumer instances
```

---

## 검증 기준 (Acceptance Criteria)

### 자동화된 검증

```java
@SpringBootTest
public class Scenario2Test {

    @Test
    public void testKafkaConsumerLagSpike() {
        // 1. 시나리오 트리거
        triggerScenario("scenario-2");

        // 2. Lag 메트릭 발생 확인
        await().atMost(60, SECONDS).until(() -> {
            // Kafka Consumer Lag 조회
            Map<TopicPartition, Long> lag = kafkaConsumerService.getCurrentLag();
            return lag.values().stream().anyMatch(l -> l > 10000);
        });

        // 3. 60초 이내 장애 감지 확인
        await().atMost(60, SECONDS).until(() -> {
            List<Incident> incidents = incidentRepository
                .findByRuleId("kafka.consumer.lag.spike");
            return !incidents.isEmpty();
        });

        Incident incident = incidents.get(0);
        assertThat(incident.getSeverity()).isIn(Severity.WARNING, Severity.CRITICAL);

        // 4. 120초 이내 AI 분석 완료 확인
        await().atMost(120, SECONDS).until(() -> {
            return analysisReportRepository
                .findByIncidentId(incident.getId())
                .isPresent();
        });

        AnalysisReport report = analysisReportRepository
            .findByIncidentId(incident.getId())
            .get();

        // 5. 근본 원인에 "throughput" 또는 "slow processing" 포함 확인
        assertThat(report.getRootCause().getSummary())
            .containsAnyOf("throughput", "slow processing", "backlog", "lag");

        // 6. 스케일링 또는 배치 처리 권장 확인
        assertThat(report.getRecommendedActions())
            .anyMatch(action ->
                action.getAction().contains("scale") ||
                action.getAction().contains("batch")
            );
    }
}
```

### 수동 검증 체크리스트

- [ ] Kafka Consumer Lag 메트릭이 10,000 초과 확인
- [ ] Slack에 CRITICAL 알림 수신
- [ ] Web UI에서 Lag 메트릭 차트 확인 (급증 패턴)
- [ ] AI 분석에서 처리 속도 불균형(throughput mismatch) 식별
- [ ] 권장 조치에 스케일링 또는 최적화 방안 포함
- [ ] 예방 방법에 모니터링/알림 설정 언급

---

## 참고 자료

### Kafka Consumer Lag 모니터링

```bash
# Kafka Consumer Group 상태 확인
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group incident-detection-group \
  --describe

# 출력 예시:
# GROUP                    TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# incident-detection-group metrics  0          12345           27579           15234
# incident-detection-group metrics  1          11000           11050           50
# incident-detection-group metrics  2          10500           10500           0
```

### Spring Kafka Configuration

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: incident-detection-group
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      max-poll-records: 500  # 한 번에 가져올 메시지 수
      fetch-min-size: 1024   # 최소 fetch 크기
    listener:
      ack-mode: manual
      concurrency: 3  # Consumer 스레드 수 (Partition 수와 매칭)
```

### 배치 처리 예시

```java
// ✅ 효율적인 배치 처리
@KafkaListener(topics = "metrics", groupId = "incident-detection-group")
public void consume(List<ConsumerRecord<String, String>> records,
                   Acknowledgment acknowledgment) {

    List<MetricEvent> events = records.stream()
        .map(record -> deserialize(record.value()))
        .collect(Collectors.toList());

    // 배치로 DB 저장 (단일 트랜잭션)
    metricRepository.saveAll(events);

    // 배치로 Rule Engine 평가
    List<Incident> incidents = ruleEngine.evaluateBatch(events);
    incidentRepository.saveAll(incidents);

    // Manual commit
    acknowledgment.acknowledge();

    log.info("Processed batch of {} messages", records.size());
}
```

### Prometheus 메트릭 수집

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kafka-lag'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 30s

# 쿼리 예시
kafka_consumer_lag{group="incident-detection-group", topic="metrics"}
rate(kafka_consumer_lag[5m]) > 0  # Lag이 증가 중인지 확인
```

---

## 실제 사례 참고

### Case Study 1: LinkedIn Kafka Lag 2018
- **원인**: Consumer 재배포 중 lag 급증
- **영향**: 실시간 피드 업데이트 4시간 지연
- **해결**: Consumer 인스턴스 2배 증가 + 배치 처리

### Case Study 2: Uber Real-time Analytics
- **원인**: ML 모델 추론 시간 증가로 처리 속도 저하
- **영향**: 실시간 분석 30분 지연
- **해결**: GPU 기반 추론 서버 도입

---

## 추가 고려사항

### Lag 계산 방식

```
Consumer Lag = Log End Offset - Current Offset

예시:
- Log End Offset: 27,579 (최신 메시지)
- Current Offset: 12,345 (현재 처리 중인 메시지)
- Lag: 15,234 messages
```

### Rebalancing 영향

Consumer Group 리밸런싱 시 일시적으로 lag 증가 가능:
- 새 Consumer 추가
- 기존 Consumer 장애
- 파티션 수 변경

AI는 리밸런싱 이벤트와 실제 성능 문제를 구분할 수 있어야 함.

### 프로덕션 권장사항

```java
// ✅ 올바른 Consumer 설정
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
        kafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3); // Partition 수와 동일
        factory.setBatchListener(true); // 배치 처리 활성화
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);

        // Lag 모니터링
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            (record, exception) -> {
                log.error("Error processing message", exception);
            }
        ));

        return factory;
    }
}
```

---

이 시나리오는 이벤트 기반 시스템에서 흔히 발생하는 처리량(throughput) 불일치 문제를 다룹니다. AI가 메트릭 패턴을 분석하여 근본 원인을 정확히 파악하고 스케일링, 배치 처리 등 실용적인 해결책을 제시하는지 평가하는 좋은 벤치마크입니다.
