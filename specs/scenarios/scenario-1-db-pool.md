# Scenario 1: Database Connection Pool Exhaustion

## 개요

**시나리오 ID**: `scenario-1`
**시나리오 명**: Database Connection Pool Exhaustion
**심각도**: CRITICAL
**예상 발생 빈도**: 중간 (월 1-2회)

### 설명
애플리케이션의 데이터베이스 커넥션 풀이 고갈되어 새로운 데이터베이스 연결을 생성할 수 없는 상황입니다. 일반적으로 커넥션 누수(connection leak) 또는 급격한 트래픽 증가로 인해 발생합니다.

### 비즈니스 영향
- 모든 데이터베이스 의존 API 요청 실패
- 사용자 로그인, 결제, 주문 등 핵심 기능 중단
- 추정 매출 손실: $50K/hour
- 사용자 불만 및 평판 손상

---

## 근본 원인 (Root Cause)

### 주요 원인

1. **Connection Leak (가장 흔함)**
   ```java
   // ❌ 잘못된 코드 예시
   public User getUser(Long id) {
       Connection conn = dataSource.getConnection();
       // ... 쿼리 실행 ...
       // conn.close() 누락!
       return user;
   }

   // ❌ 예외 발생 시 누수
   public User getUser(Long id) {
       Connection conn = dataSource.getConnection();
       try {
           // 쿼리 실행
       } catch (SQLException e) {
           // 예외 처리
           // conn.close() 누락!
       }
       return user;
   }
   ```

2. **Long-Running Transactions**
   - 커넥션을 오래 점유하는 복잡한 트랜잭션
   - 배치 작업이 커넥션을 장시간 보유

3. **Insufficient Pool Size**
   - 트래픽 증가에 비해 풀 크기 부족
   - 잘못된 용량 계획

4. **Database Performance Issue**
   - Slow query로 인한 커넥션 점유 시간 증가
   - Database lock/deadlock

---

## 시뮬레이션 방법

### API 트리거

```bash
curl -X POST http://localhost:8080/api/v1/scenarios/scenario-1/trigger \
  -H "Content-Type: application/json" \
  -d '{
    "duration": "PT5M",
    "intensity": "HIGH"
  }'
```

### 시뮬레이션 로직

```java
@Service
public class DbPoolExhaustionScenario {

    @Autowired
    private DataSource dataSource;

    public void execute(String intensity) {
        int leakCount = intensity.equals("HIGH") ? 45 : 30;
        List<Connection> leakedConnections = new ArrayList<>();

        // 의도적으로 커넥션을 close하지 않고 보유
        for (int i = 0; i < leakCount; i++) {
            try {
                Connection conn = dataSource.getConnection();
                // Slow query 시뮬레이션
                Statement stmt = conn.createStatement();
                stmt.execute("SELECT SLEEP(10)"); // 10초 대기

                // ❌ 의도적으로 close 하지 않음
                leakedConnections.add(conn);

                Thread.sleep(1000); // 1초 간격
            } catch (Exception e) {
                // 커넥션 풀 고갈 발생
                break;
            }
        }

        // 5분 후 정리 (시나리오 종료)
        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                leakedConnections.forEach(conn -> {
                    try { conn.close(); } catch (Exception e) {}
                });
            }
        }, 300000); // 5분
    }
}
```

### 생성되는 메트릭 데이터

Kafka `metrics` topic에 다음 메시지 발행:

```json
{
  "timestamp": "2026-02-14T14:30:12.345Z",
  "service": "payment-service",
  "host": "payment-prod-01",
  "metric": "db.connection.pool.usage",
  "value": 98.5,
  "unit": "%",
  "metadata": {
    "active": 49,
    "idle": 0,
    "max": 50,
    "pending": 127
  }
}
```

### 생성되는 로그 데이터

Kafka `logs` topic에 다음 메시지 발행:

```json
{
  "timestamp": "2026-02-14T14:30:12.345Z",
  "level": "ERROR",
  "service": "payment-service",
  "host": "payment-prod-01",
  "logger": "com.zaxxer.hikari.pool.HikariPool",
  "message": "HikariPool-1 - Connection is not available, request timed out after 30000ms.",
  "exception": "java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available\n\tat com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:186)\n\tat com.payment.service.UserService.getUserDetails(UserService.java:45)",
  "context": {
    "requestId": "req-abc-123",
    "userId": "12345"
  }
}
```

---

## 감지 기준 (Detection Rule)

### 규칙 정의

```java
@Component
public class DbConnectionPoolRule implements DetectionRule {

    @Override
    public boolean evaluate(MetricEvent event) {
        return event.getMetricName().equals("db.connection.pool.usage")
            && event.getValue() > 90.0
            && event.getDuration() > Duration.ofMinutes(2);
    }

    @Override
    public Severity getSeverity() {
        return Severity.CRITICAL;
    }

    @Override
    public String getDescription() {
        return "Database connection pool usage > 90% for 2 minutes";
    }
}
```

### 임계값

| 메트릭 | 임계값 | 지속 시간 | 심각도 |
|--------|--------|-----------|--------|
| db.connection.pool.usage | > 90% | 2분 | CRITICAL |
| db.connection.pool.usage | > 80% | 5분 | WARNING |

### 예상 감지 시간

- **목표**: 60초 이내
- **프로세스**:
  1. 메트릭 수집 (Kafka) - 즉시
  2. Rule Engine 평가 - 1초 이내
  3. Incident 생성 - 1초 이내
  4. **총 소요 시간**: ~10초

---

## AI 분석 예상 결과

### Root Cause Analysis

```json
{
  "root_cause": {
    "summary": "Connection leak in UserService.getUserDetails()",
    "details": "The method UserService.getUserDetails() fails to close database connections in the exception handler path. When exceptions occur (e.g., SQLException), the connection.close() is not called, causing gradual pool exhaustion over time.",
    "confidence": 0.95,
    "evidence": [
      "Error logs showing 'Connection is not available' from HikariPool",
      "Stack trace pointing to UserService.getUserDetails() line 45",
      "Connection pool usage steadily increasing from 30% to 98% over 30 minutes"
    ]
  }
}
```

### Impact Assessment

```json
{
  "impact": {
    "affected_services": ["payment-service", "order-service"],
    "estimated_users": 1500,
    "severity": "CRITICAL",
    "business_impact": "Payment processing blocked. Estimated revenue loss: $50K/hour",
    "downstream_effects": [
      "Order creation failing",
      "User authentication timing out",
      "Admin dashboard inaccessible"
    ]
  }
}
```

### Recommended Actions

```json
{
  "recommended_actions": [
    {
      "priority": 1,
      "action": "Restart payment-service immediately to release leaked connections",
      "expected_result": "Immediate recovery of service availability",
      "estimated_duration": "PT2M",
      "risk": "LOW"
    },
    {
      "priority": 2,
      "action": "Deploy hotfix PR#1234 to add try-with-resources in UserService.getUserDetails()",
      "expected_result": "Prevent recurrence of connection leaks",
      "estimated_duration": "PT1H",
      "risk": "LOW"
    },
    {
      "priority": 3,
      "action": "Increase connection pool size from 50 to 100 as temporary mitigation",
      "expected_result": "More headroom for traffic spikes",
      "estimated_duration": "PT15M",
      "risk": "MEDIUM - Increased database load"
    },
    {
      "priority": 4,
      "action": "Add connection pool monitoring alert at 80% threshold",
      "expected_result": "Early warning before exhaustion",
      "estimated_duration": "PT30M",
      "risk": "LOW"
    }
  ]
}
```

### Prevention

```
1. Code Review: Enforce try-with-resources for all database operations
2. Static Analysis: Add SonarQube rule to detect potential connection leaks
3. Load Testing: Regular load tests to identify connection pool sizing issues
4. Monitoring: Set up alerts at 70%, 80%, 90% pool usage levels
5. Circuit Breaker: Implement circuit breaker pattern for database calls
```

---

## 검증 기준 (Acceptance Criteria)

### 자동화된 검증

```java
@SpringBootTest
@AutoConfigureMockMvc
public class Scenario1Test {

    @Test
    public void testDatabaseConnectionPoolExhaustion() {
        // 1. 시나리오 트리거
        triggerScenario("scenario-1");

        // 2. 60초 이내 장애 감지 확인
        await().atMost(60, SECONDS).until(() -> {
            List<Incident> incidents = incidentRepository
                .findByServiceNameAndRuleId("payment-service", "db.connection.pool.exhausted");
            return !incidents.isEmpty();
        });

        Incident incident = incidents.get(0);
        assertThat(incident.getSeverity()).isEqualTo(Severity.CRITICAL);

        // 3. 120초 이내 AI 분석 완료 확인
        await().atMost(120, SECONDS).until(() -> {
            return analysisReportRepository
                .findByIncidentId(incident.getId())
                .isPresent();
        });

        AnalysisReport report = analysisReportRepository
            .findByIncidentId(incident.getId())
            .get();

        // 4. 근본 원인 정확성 검증
        assertThat(report.getRootCause().getSummary())
            .containsIgnoringCase("connection leak");
        assertThat(report.getRootCause().getConfidence())
            .isGreaterThan(0.8);

        // 5. 권장 조치 존재 확인
        assertThat(report.getRecommendedActions())
            .hasSizeGreaterThan(2);
        assertThat(report.getRecommendedActions().get(0).getAction())
            .containsIgnoringCase("restart");
    }
}
```

### 수동 검증 체크리스트

- [ ] Slack에 CRITICAL 알림 수신 (60초 이내)
- [ ] Web UI Dashboard에 장애 표시
- [ ] Incident Detail 페이지에서 메트릭 그래프 확인
- [ ] AI 분석 리포트가 읽기 쉽고 actionable한지 확인
- [ ] Slack에 AI 분석 완료 알림 수신 (120초 이내)
- [ ] 제안된 조치 사항이 실제로 문제 해결에 도움이 되는지 평가

---

## 참고 자료

### HikariCP Configuration

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 30000  # 30초
      idle-timeout: 600000       # 10분
      max-lifetime: 1800000      # 30분
      leak-detection-threshold: 60000  # 60초 (leak 감지)
```

### 모니터링 쿼리

```sql
-- 현재 활성 커넥션 수 조회
SELECT COUNT(*)
FROM information_schema.processlist
WHERE db = 'incident_analysis';

-- Long-running queries 조회
SELECT id, user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE command != 'Sleep' AND time > 10
ORDER BY time DESC;
```

### 유용한 명령어

```bash
# HikariCP 메트릭 확인
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending

# 시나리오 실행 중 메트릭 모니터링
watch -n 1 'curl -s http://localhost:8080/actuator/metrics/hikaricp.connections.usage | jq'
```

---

## 실제 사례 참고

### Case Study 1: Reddit 2020 Incident
- **원인**: Connection pool 크기 부족 + 느린 쿼리
- **영향**: 30분간 서비스 다운
- **해결**: Pool 크기 증가 + 쿼리 최적화

### Case Study 2: Shopify Black Friday 2019
- **원인**: 급격한 트래픽 증가로 pool exhaustion
- **영향**: 체크아웃 지연
- **해결**: Auto-scaling + Connection pool tuning

---

## 추가 고려사항

### Edge Cases

1. **Partial Pool Exhaustion**
   - 일부 커넥션만 누수되어 간헐적 장애 발생
   - 감지가 어려워 장시간 지속 가능

2. **Database Restart 후 Pool 재생성**
   - 기존 커넥션 invalid 상태
   - Pool validation 로직 필요

3. **Read Replica Connection Pool**
   - Master/Replica 각각 별도 pool
   - Replica pool exhaustion은 덜 심각

### 프로덕션 권장사항

```java
// ✅ 올바른 커넥션 관리
public User getUser(Long id) {
    // try-with-resources 사용
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement(
             "SELECT * FROM users WHERE id = ?")) {

        stmt.setLong(1, id);
        ResultSet rs = stmt.executeQuery();

        if (rs.next()) {
            return mapUser(rs);
        }
        return null;
    } catch (SQLException e) {
        log.error("Database error", e);
        throw new DatabaseException(e);
    }
    // 자동으로 conn.close() 호출됨
}

// 또는 Spring JdbcTemplate 사용 (권장)
public User getUser(Long id) {
    return jdbcTemplate.queryForObject(
        "SELECT * FROM users WHERE id = ?",
        new Object[]{id},
        new UserRowMapper()
    );
}
```

---

이 시나리오는 가장 흔하게 발생하는 프로덕션 장애 중 하나이므로, AI 도구가 얼마나 정확하게 근본 원인을 식별하고 실용적인 조치를 제안하는지 평가하는 좋은 벤치마크입니다.
