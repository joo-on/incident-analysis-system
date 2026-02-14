# Scenario 4: Cascading Service Failure

## 개요

**시나리오 ID**: `scenario-4`
**시나리오 명**: Cascading Service Failure
**심각도**: CRITICAL
**예상 발생 빈도**: 낮음 (분기 1회)

### 설명
하나의 다운스트림 서비스 장애가 업스트림 서비스들로 연쇄적으로 전파되어 시스템 전체에 영향을 미치는 상황입니다. Circuit breaker, timeout, bulkhead 등의 방어 메커니즘이 없을 때 발생합니다.

### 비즈니스 영향
- 전체 시스템 마비 (도미노 효과)
- 모든 사용자 기능 중단
- 막대한 매출 손실
- 복구 시간 증가 (여러 서비스 동시 복구 필요)

---

## 근본 원인 (Root Cause)

### 주요 원인

1. **Missing Timeout Configuration**
   ```java
   // ❌ Timeout 없는 HTTP 호출
   @Service
   public class PaymentService {
       private final RestTemplate restTemplate;

       public PaymentResult processPayment(Payment payment) {
           // Timeout 설정 없음!
           // 다운스트림 서비스가 응답하지 않으면 영원히 대기
           ResponseEntity<PaymentResult> response =
               restTemplate.postForEntity(
                   "http://payment-gateway/api/charge",
                   payment,
                   PaymentResult.class
               );
           return response.getBody();
       }
   }
   ```

2. **No Circuit Breaker**
   ```java
   // ❌ Circuit Breaker 없음
   public Order createOrder(OrderRequest request) {
       // Inventory 서비스가 다운되어도 계속 호출 시도
       InventoryResponse inventory = inventoryService.reserve(request.items);

       // Payment 서비스도 타임아웃 걸림
       PaymentResult payment = paymentService.charge(request.payment);

       // Thread pool exhaustion
       return saveOrder(inventory, payment);
   }
   ```

3. **Shared Thread Pool**
   ```java
   // ❌ 모든 외부 호출이 동일한 스레드 풀 사용
   @Bean
   public RestTemplate restTemplate() {
       return new RestTemplate();
       // Bulkhead 격리 없음
       // 한 서비스 장애가 전체 스레드 풀 고갈
   }
   ```

4. **Retry Storm**
   ```java
   // ❌ 무한 재시도
   @Retryable(maxAttempts = Integer.MAX_VALUE)
   public User getUser(Long id) {
       return userService.findById(id);
   }
   ```

5. **Synchronous Dependencies**
   - 비동기 처리 없이 동기적으로 다운스트림 서비스 의존
   - 서비스 간 강결합

---

## 시뮬레이션 방법

### API 트리거

```bash
curl -X POST http://localhost:8080/api/v1/scenarios/scenario-4/trigger \
  -H "Content-Type: application/json" \
  -d '{
    "duration": "PT5M",
    "intensity": "HIGH"
  }'
```

### 시뮬레이션 로직

```java
@Service
public class CascadingFailureScenario {

    @Autowired
    private RestTemplate restTemplate;

    public void execute(String intensity) {
        // 1. Mock 다운스트림 서비스 시작 (느린 응답)
        startSlowDownstreamService();

        // 2. 업스트림 서비스에 부하 발생
        ExecutorService executor = Executors.newFixedThreadPool(50);

        for (int i = 0; i < 100; i++) {
            executor.submit(() -> {
                try {
                    // Timeout 없는 호출 → 스레드 블로킹
                    callDownstreamServiceWithoutTimeout();
                } catch (Exception e) {
                    log.error("Request failed", e);
                }
            });
        }

        // 3. 연쇄 장애 시뮬레이션
        simulateCascadingFailure(intensity);
    }

    private void startSlowDownstreamService() {
        // Mock server: 응답하지 않거나 매우 느림
        MockRestServiceServer mockServer = MockRestServiceServer
            .createServer(restTemplate);

        mockServer.expect(requestTo("http://payment-gateway/api/charge"))
            .andRespond(request -> {
                Thread.sleep(60000); // 60초 대기 (timeout)
                return new MockClientHttpResponse(new byte[0], HttpStatus.OK);
            });
    }

    private void simulateCascadingFailure(String intensity) {
        /*
         * 장애 전파 시나리오:
         *
         * 1. Payment Gateway 다운 (의도적)
         *    ↓
         * 2. Payment Service 스레드 블로킹 (timeout 없음)
         *    ↓
         * 3. Payment Service 스레드 풀 고갈
         *    ↓
         * 4. Order Service가 Payment Service 호출 실패
         *    ↓
         * 5. Order Service 스레드 풀 고갈
         *    ↓
         * 6. Frontend API Gateway 호출 실패
         *    ↓
         * 7. 전체 시스템 마비
         */

        // Phase 1: Payment Gateway 장애 (t=0)
        shutdownService("payment-gateway");

        // Phase 2: Payment Service 스레드 고갈 (t=30s)
        scheduleThreadPoolExhaustion("payment-service", 30);

        // Phase 3: Order Service 영향 (t=60s)
        scheduleServiceDegradation("order-service", 60);

        // Phase 4: API Gateway 장애 (t=90s)
        scheduleServiceDegradation("api-gateway", 90);
    }
}
```

### 생성되는 메트릭 데이터

**Phase 1: Payment Gateway 다운**
```json
{
  "timestamp": "2026-02-14T14:30:00.000Z",
  "service": "payment-gateway",
  "host": "payment-gw-01",
  "metric": "http.server.requests",
  "value": 0,
  "unit": "requests/sec",
  "metadata": {
    "status": "DOWN",
    "errorRate": 100.0
  }
}
```

**Phase 2: Payment Service 스레드 고갈**
```json
{
  "timestamp": "2026-02-14T14:30:30.000Z",
  "service": "payment-service",
  "host": "payment-01",
  "metric": "http.server.requests.active",
  "value": 200,
  "unit": "threads",
  "metadata": {
    "max": 200,
    "usage": 100.0,
    "pendingRequests": 1523
  }
}
```

**Phase 3: Order Service 영향**
```json
{
  "timestamp": "2026-02-14T14:31:00.000Z",
  "service": "order-service",
  "host": "order-01",
  "metric": "http.client.requests.timeout",
  "value": 45,
  "unit": "count/min",
  "metadata": {
    "target": "payment-service",
    "timeoutDuration": "PT30S"
  }
}
```

### 생성되는 로그 데이터

**Payment Service**
```json
{
  "timestamp": "2026-02-14T14:30:15.000Z",
  "level": "ERROR",
  "service": "payment-service",
  "host": "payment-01",
  "logger": "org.springframework.web.client.RestTemplate",
  "message": "I/O error on POST request for \"http://payment-gateway/api/charge\": Read timed out",
  "exception": "java.net.SocketTimeoutException: Read timed out\n\tat java.net.SocketInputStream.socketRead0(Native Method)",
  "context": {
    "requestId": "req-123",
    "downstream": "payment-gateway",
    "timeout": "NOT_SET"
  }
}
```

**Order Service**
```json
{
  "timestamp": "2026-02-14T14:31:05.000Z",
  "level": "ERROR",
  "service": "order-service",
  "host": "order-01",
  "logger": "com.order.service.OrderService",
  "message": "Failed to create order: Timeout calling payment-service",
  "exception": "org.springframework.web.client.ResourceAccessException: I/O error on POST request",
  "context": {
    "orderId": "ORD-12345",
    "userId": "user-789",
    "downstream": "payment-service",
    "retryAttempt": 3
  }
}
```

**API Gateway**
```json
{
  "timestamp": "2026-02-14T14:31:30.000Z",
  "level": "CRITICAL",
  "service": "api-gateway",
  "host": "gateway-01",
  "logger": "com.gateway.HealthCheck",
  "message": "Multiple downstream services unhealthy: payment-service, order-service",
  "context": {
    "unhealthyServices": ["payment-service", "order-service"],
    "totalServices": 12,
    "healthyServices": 10
  }
}
```

---

## 감지 기준 (Detection Rule)

### 규칙 정의

```java
@Component
public class CascadingFailureRule implements DetectionRule {

    @Autowired
    private IncidentRepository incidentRepository;

    @Override
    public boolean evaluate(MetricEvent event) {
        // 1. 여러 서비스에서 동시에 장애 발생 감지
        Instant fiveMinutesAgo = Instant.now().minus(5, ChronoUnit.MINUTES);
        List<Incident> recentIncidents = incidentRepository
            .findBySeverityAndDetectedAtAfter(Severity.CRITICAL, fiveMinutesAgo);

        // 2. 5분 내 3개 이상의 서로 다른 서비스에서 CRITICAL 장애
        Set<String> affectedServices = recentIncidents.stream()
            .map(Incident::getServiceName)
            .collect(Collectors.toSet());

        if (affectedServices.size() >= 3) {
            return true;
        }

        // 3. 특정 패턴: Thread Pool Exhaustion + Downstream Timeout
        boolean hasThreadPoolIssue = recentIncidents.stream()
            .anyMatch(i -> i.getRuleId().contains("thread.pool"));

        boolean hasTimeoutIssue = recentIncidents.stream()
            .anyMatch(i -> i.getRuleId().contains("timeout"));

        return hasThreadPoolIssue && hasTimeoutIssue;
    }

    @Override
    public Severity getSeverity() {
        return Severity.CRITICAL;
    }

    @Override
    public String getDescription() {
        return "Cascading failure detected: 3+ services affected in 5 minutes";
    }
}
```

### 임계값

| 조건 | 기준 | 심각도 |
|------|------|--------|
| 동시 다발 장애 | 5분 내 3개 이상 서비스 CRITICAL | CRITICAL |
| 연쇄 패턴 | Thread Pool + Timeout 동시 발생 | CRITICAL |
| Error Rate | 여러 서비스 에러율 > 50% | CRITICAL |

---

## AI 분석 예상 결과

### Root Cause Analysis

```json
{
  "root_cause": {
    "summary": "Payment Gateway outage triggered cascading failure due to missing circuit breakers and timeouts",
    "details": "The Payment Gateway service became unresponsive at 14:30:00. Because Payment Service had no timeout configured for downstream calls, threads began blocking indefinitely. Within 30 seconds, all 200 threads in the Payment Service pool were exhausted. Order Service, which depends on Payment Service, then experienced timeouts on its calls. Without circuit breakers, Order Service continued retrying failed requests, leading to its own thread pool exhaustion. This cascaded to API Gateway, causing system-wide failure.",
    "confidence": 0.96,
    "evidence": [
      "Payment Gateway health check failed at 14:30:00",
      "Payment Service thread pool reached 100% usage at 14:30:30",
      "Order Service timeout errors increased from 0 to 45/min at 14:31:00",
      "API Gateway error rate spiked from 0.1% to 87% at 14:31:30",
      "No circuit breaker open events logged",
      "RestTemplate configured without timeout settings"
    ],
    "failure_chain": [
      {
        "order": 1,
        "service": "payment-gateway",
        "time": "14:30:00",
        "issue": "Service unresponsive (root cause)"
      },
      {
        "order": 2,
        "service": "payment-service",
        "time": "14:30:30",
        "issue": "Thread pool exhaustion due to blocked calls"
      },
      {
        "order": 3,
        "service": "order-service",
        "time": "14:31:00",
        "issue": "Timeouts calling payment-service, retry storm"
      },
      {
        "order": 4,
        "service": "api-gateway",
        "time": "14:31:30",
        "issue": "Widespread failures, system-wide impact"
      }
    ]
  }
}
```

### Impact Assessment

```json
{
  "impact": {
    "affected_services": [
      "payment-gateway",
      "payment-service",
      "order-service",
      "api-gateway"
    ],
    "severity": "CRITICAL",
    "business_impact": "Complete system outage. All customer-facing features unavailable. Estimated revenue loss: $200K/hour. 15,000+ active users affected.",
    "blast_radius": {
      "total_services": 12,
      "affected_services": 4,
      "percentage": 33.3
    },
    "downstream_effects": [
      "Payment processing blocked",
      "Order creation failed",
      "User login timing out",
      "Admin dashboard inaccessible",
      "Mobile app unusable"
    ],
    "recovery_complexity": "HIGH - Multiple services need coordinated restart"
  }
}
```

### Recommended Actions

```json
{
  "recommended_actions": [
    {
      "priority": 1,
      "action": "Restart payment-gateway service to restore root service",
      "expected_result": "Remove root cause, allow dependent services to recover",
      "estimated_duration": "PT3M",
      "risk": "LOW"
    },
    {
      "priority": 2,
      "action": "Rolling restart of payment-service, order-service, api-gateway to clear blocked threads",
      "expected_result": "Release hung threads, restore service availability",
      "estimated_duration": "PT10M",
      "risk": "MEDIUM - Coordinate restarts to avoid further disruption"
    },
    {
      "priority": 3,
      "action": "Deploy circuit breaker hotfix using Resilience4j",
      "code_example": "@CircuitBreaker(name = \"payment\", fallbackMethod = \"paymentFallback\")\npublic PaymentResult processPayment(Payment payment) {\n  return paymentGateway.charge(payment);\n}\n\npublic PaymentResult paymentFallback(Payment payment, Exception e) {\n  return PaymentResult.failed(\"Payment service unavailable\");\n}",
      "expected_result": "Prevent future cascading failures, contain blast radius",
      "estimated_duration": "PT2H",
      "risk": "LOW"
    },
    {
      "priority": 4,
      "action": "Configure RestTemplate with timeouts (connect: 2s, read: 10s)",
      "code_example": "@Bean\npublic RestTemplate restTemplate() {\n  HttpComponentsClientHttpRequestFactory factory = \n    new HttpComponentsClientHttpRequestFactory();\n  factory.setConnectTimeout(2000);\n  factory.setReadTimeout(10000);\n  return new RestTemplate(factory);\n}",
      "expected_result": "Prevent indefinite thread blocking",
      "estimated_duration": "PT1H",
      "risk": "LOW"
    },
    {
      "priority": 5,
      "action": "Implement bulkhead pattern: separate thread pools per downstream service",
      "expected_result": "Isolate failures, one service issue doesn't exhaust all threads",
      "estimated_duration": "PT4H",
      "risk": "MEDIUM - Requires thread pool tuning"
    }
  ]
}
```

### Prevention

```
1. Circuit Breakers: Implement Resilience4j circuit breakers for all external calls
2. Timeouts: Enforce connect/read timeouts on all HTTP clients (default: 2s/10s)
3. Bulkheads: Use separate thread pools for different downstream services
4. Retry Policies: Exponential backoff with max 3 retries, circuit breaker integration
5. Chaos Engineering: Regular chaos tests (e.g., shut down random services)
6. Observability: Distributed tracing (Zipkin/Jaeger) to visualize cascading failures
7. Load Shedding: Implement rate limiting and queue-based load shedding
8. Health Checks: Comprehensive health checks including downstream dependencies
```

---

## 검증 기준 (Acceptance Criteria)

### 자동화된 검증

```java
@SpringBootTest
public class Scenario4Test {

    @Test
    public void testCascadingServiceFailure() {
        // 1. 시나리오 트리거
        triggerScenario("scenario-4");

        // 2. 다운스트림 서비스 장애 발생 확인
        await().atMost(30, SECONDS).until(() -> {
            return !isServiceHealthy("payment-gateway");
        });

        // 3. 연쇄 장애 전파 확인 (60초 이내)
        await().atMost(60, SECONDS).until(() -> {
            List<String> unhealthyServices = getUnhealthyServices();
            return unhealthyServices.size() >= 3;
        });

        // 4. Cascading Failure 장애 감지 확인
        await().atMost(60, SECONDS).until(() -> {
            List<Incident> incidents = incidentRepository
                .findByRuleId("cascading.failure");
            return !incidents.isEmpty();
        });

        Incident incident = incidents.get(0);
        assertThat(incident.getSeverity()).isEqualTo(Severity.CRITICAL);

        // 5. AI 분석 완료 확인
        await().atMost(120, SECONDS).until(() -> {
            return analysisReportRepository
                .findByIncidentId(incident.getId())
                .isPresent();
        });

        AnalysisReport report = analysisReportRepository
            .findByIncidentId(incident.getId())
            .get();

        // 6. 근본 원인에 "cascading" 또는 "chain" 포함
        assertThat(report.getRootCause().getSummary())
            .containsAnyOf("cascading", "chain", "propagat", "downstream");

        // 7. Failure chain 식별 확인
        assertThat(report.getRootCause().getDetails())
            .contains("payment-gateway")
            .contains("payment-service")
            .contains("order-service");

        // 8. Circuit breaker 또는 timeout 권장 확인
        assertThat(report.getRecommendedActions())
            .anyMatch(action ->
                action.getAction().containsAnyOf(
                    "circuit breaker",
                    "timeout",
                    "bulkhead",
                    "resilience"
                )
            );
    }
}
```

### 수동 검증 체크리스트

- [ ] 3개 이상 서비스에서 동시 장애 발생 확인
- [ ] Slack에 CRITICAL 알림 수신
- [ ] Web UI Dashboard에 여러 서비스 장애 표시
- [ ] AI 분석에서 장애 전파 순서(failure chain) 식별
- [ ] 근본 원인으로 Circuit Breaker/Timeout 부재 지적
- [ ] 권장 조치에 Resilience4j 또는 유사 라이브러리 제안
- [ ] 코드 예시 포함 여부 확인

---

## 참고 자료

### Resilience4j Circuit Breaker 설정

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      payment:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
        recordExceptions:
          - java.net.SocketTimeoutException
          - java.io.IOException
  timelimiter:
    instances:
      payment:
        timeoutDuration: 10s
  retry:
    instances:
      payment:
        maxAttempts: 3
        waitDuration: 1s
        exponentialBackoffMultiplier: 2
```

### Circuit Breaker 사용 예시

```java
@Service
public class PaymentService {

    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    @TimeLimiter(name = "payment")
    @Retry(name = "payment")
    public CompletableFuture<PaymentResult> processPayment(Payment payment) {
        return CompletableFuture.supplyAsync(() ->
            restTemplate.postForObject(
                "http://payment-gateway/api/charge",
                payment,
                PaymentResult.class
            )
        );
    }

    // Fallback 메서드
    private CompletableFuture<PaymentResult> paymentFallback(
        Payment payment, Exception e) {

        log.warn("Payment service unavailable, using fallback", e);

        // Graceful degradation
        return CompletableFuture.completedFuture(
            PaymentResult.builder()
                .status(PaymentStatus.PENDING)
                .message("Payment processing delayed, will retry")
                .build()
        );
    }
}
```

### Timeout 설정

```java
@Bean
public RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory factory =
        new HttpComponentsClientHttpRequestFactory();

    // Connection timeout: 서버 연결까지 대기 시간
    factory.setConnectTimeout(2000);  // 2초

    // Read timeout: 응답 읽기까지 대기 시간
    factory.setReadTimeout(10000);    // 10초

    return new RestTemplate(factory);
}
```

### Bulkhead Pattern

```java
@Configuration
public class BulkheadConfig {

    @Bean
    public Executor paymentExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("payment-");
        executor.initialize();
        return executor;
    }

    @Bean
    public Executor orderExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("order-");
        executor.initialize();
        return executor;
    }
}

@Service
public class PaymentService {
    @Async("paymentExecutor")  // 전용 스레드 풀 사용
    public CompletableFuture<PaymentResult> processPayment(Payment payment) {
        // ...
    }
}
```

---

## 실제 사례 참고

### Case Study 1: AWS S3 Outage 2017
- **원인**: S3 서비스 장애
- **영향**: S3에 의존하는 수많은 서비스 연쇄 다운
- **교훈**: 단일 장애 지점(SPOF) 제거, Circuit Breaker 필수

### Case Study 2: GitHub Outage 2020
- **원인**: 데이터베이스 장애
- **영향**: 전체 GitHub 서비스 5시간 다운
- **해결**: Database connection pooling 개선

---

## 추가 고려사항

### Distributed Tracing

Zipkin 또는 Jaeger로 분산 추적:

```java
@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder
        .interceptors(new TracingClientHttpRequestInterceptor(tracer))
        .build();
}
```

Trace를 통해 장애 전파 경로 시각화:

```
api-gateway [200ms]
  └─> order-service [150ms]
      └─> payment-service [TIMEOUT 30s]
          └─> payment-gateway [NO RESPONSE]
```

### Health Check 계층화

```java
@Component
public class CompositeHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        Health.Builder builder = Health.up();

        // 자체 상태
        if (!isDatabaseHealthy()) {
            builder.down().withDetail("database", "Connection failed");
        }

        // 다운스트림 서비스 상태
        if (!isPaymentGatewayHealthy()) {
            builder.down().withDetail("payment-gateway", "Unreachable");
        }

        return builder.build();
    }
}
```

### Chaos Engineering

Netflix Chaos Monkey 스타일 테스트:

```bash
# 랜덤 서비스 종료
chaos kill --service=payment-gateway --probability=0.1

# 네트워크 지연 주입
chaos latency --service=order-service --delay=5s

# 에러율 주입
chaos error --service=payment-service --rate=0.5
```

---

이 시나리오는 마이크로서비스 아키텍처에서 가장 위험한 연쇄 장애를 다룹니다. AI가 여러 서비스 간의 의존 관계를 파악하고, 장애 전파 경로를 추적하며, Circuit Breaker, Timeout, Bulkhead 등 구체적인 방어 메커니즘을 제안하는지 평가하는 고급 벤치마크입니다.
