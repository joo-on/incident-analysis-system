# System Architecture

## 1. Architecture Overview

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     External Systems                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │   App    │  │   DB     │  │  Cache   │  │   API    │        │
│  │  Servers │  │ Servers  │  │ Servers  │  │ Servers  │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
│       │             │             │             │               │
│       └─────────────┴─────────────┴─────────────┘               │
│                         │ (Metrics & Logs)                      │
└─────────────────────────┼─────────────────────────────────────┘
                          ▼
         ┌────────────────────────────────┐
         │       Apache Kafka             │
         │  ┌──────────┐  ┌──────────┐   │
         │  │ metrics  │  │   logs   │   │
         │  │  topic   │  │  topic   │   │
         │  └──────────┘  └──────────┘   │
         └────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│              Incident Analysis System                            │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   Detection Engine                         │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │  │
│  │  │   Kafka      │  │    Rule      │  │  Incident    │    │  │
│  │  │  Consumer    │─▶│   Engine     │─▶│  Generator   │    │  │
│  │  └──────────────┘  └──────────────┘  └──────┬───────┘    │  │
│  └──────────────────────────────────────────────┼────────────┘  │
│                                                  │               │
│  ┌───────────────────────────────────────────────┼────────────┐ │
│  │                  Spring Boot API               │            │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────▼───────┐   │ │
│  │  │   Incident   │  │      AI      │  │  Notification  │   │ │
│  │  │   Service    │◀─│   Analyzer   │─▶│    Service     │   │ │
│  │  └──────┬───────┘  └──────┬───────┘  └────────┬───────┘   │ │
│  │         │                  │                   │            │ │
│  │  ┌──────▼───────┐  ┌──────▼───────┐  ┌────────▼───────┐   │ │
│  │  │     REST     │  │  Repository  │  │     Slack      │   │ │
│  │  │  Controller  │  │    Layer     │  │     Client     │   │ │
│  │  └──────┬───────┘  └──────┬───────┘  └────────────────┘   │ │
│  └─────────┼──────────────────┼──────────────────────────────┘ │
│            │                  │                                 │
└────────────┼──────────────────┼─────────────────────────────────┘
             │                  │
             ▼                  ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │   Web UI     │    │    MySQL     │    │    Redis     │
   │ (React/Vue)  │    │  (Primary)   │    │   (Cache)    │
   └──────────────┘    └──────────────┘    └──────────────┘
             │
             ▼
   ┌──────────────┐    ┌──────────────┐
   │  Claude API  │    │   Slack      │
   │  (OpenAI)    │    │   Webhook    │
   └──────────────┘    └──────────────┘
```

### 1.2 Architecture Style

- **Monolithic Architecture**: 초기 버전은 단일 Spring Boot 애플리케이션
- **Event-Driven**: Kafka 기반 비동기 이벤트 처리
- **Layered Architecture**: Presentation, Service, Repository 계층 분리
- **Cache-Aside Pattern**: Redis를 통한 읽기 성능 최적화

## 2. Component Design

### 2.1 Detection Engine

#### 2.1.1 Kafka Consumer
**책임**:
- Kafka topics (`metrics`, `logs`)에서 메시지 소비
- 메시지 역직렬화 (JSON → Java Object)
- Consumer Group 기반 병렬 처리

**기술 스택**:
- Spring Kafka (`spring-kafka`)
- Consumer Group: `incident-detection-group`
- Offset Management: Auto-commit disabled, manual commit

**구현 상세**:
```java
@KafkaListener(
    topics = {"metrics", "logs"},
    groupId = "incident-detection-group",
    concurrency = "3"
)
public void consume(ConsumerRecord<String, String> record) {
    // 메시지 처리
    MetricEvent event = deserialize(record.value());
    ruleEngine.evaluate(event);
    // Manual commit
    acknowledgment.acknowledge();
}
```

#### 2.1.2 Rule Engine
**책임**:
- 사전 정의된 감지 규칙 평가
- 임계값 기반 장애 판단
- 중복 이벤트 필터링

**규칙 정의 방식**:
```java
public interface DetectionRule {
    boolean evaluate(MetricEvent event);
    Severity getSeverity();
    String getDescription();
}

@Component
public class CpuUsageRule implements DetectionRule {
    @Override
    public boolean evaluate(MetricEvent event) {
        return event.getMetricName().equals("cpu.usage")
            && event.getValue() > 90.0
            && event.getDuration() > Duration.ofMinutes(5);
    }

    @Override
    public Severity getSeverity() {
        return Severity.WARNING;
    }
}
```

**규칙 목록** (requirements.md FR-1 참조):
- `CpuUsageRule`
- `MemoryUsageRule`
- `DbConnectionPoolRule`
- `KafkaConsumerLagRule`
- `HttpErrorRateRule`
- `ResponseTimeRule`

**중복 제거 로직**:
- Redis Sorted Set 사용
- Key: `incident:dedup:{service}:{metric}`
- Score: 발생 timestamp
- TTL: 5분
- 동일 키 존재 시 기존 incident에 카운트만 증가

#### 2.1.3 Incident Generator
**책임**:
- 규칙 위반 시 Incident 엔티티 생성
- MySQL에 저장
- CRITICAL 장애 시 AI Analyzer에 이벤트 발행

**Incident Entity**:
```java
@Entity
@Table(name = "incidents")
public class Incident {
    @Id
    private UUID id;

    private Instant detectedAt;

    @Enumerated(EnumType.STRING)
    private Severity severity; // CRITICAL, WARNING, INFO

    @Enumerated(EnumType.STRING)
    private Status status; // OPEN, INVESTIGATING, RESOLVED

    private String serviceName;
    private String hostName;
    private String ruleId;

    @Column(columnDefinition = "JSON")
    private String metricData; // 원시 메트릭 (JSON)

    private Integer duplicateCount = 0;

    @OneToOne(mappedBy = "incident", cascade = CascadeType.ALL)
    private AnalysisReport analysisReport;
}
```

### 2.2 Spring Boot API Server

#### 2.2.1 REST Controllers

**Incident Controller**:
```
GET    /api/v1/incidents               # 장애 목록 조회 (페이징, 필터링)
GET    /api/v1/incidents/{id}          # 장애 상세 조회
PATCH  /api/v1/incidents/{id}/status   # 장애 상태 변경
GET    /api/v1/incidents/{id}/report   # AI 분석 리포트 조회
POST   /api/v1/incidents/{id}/analyze  # 수동 AI 분석 트리거
```

**Log Controller**:
```
GET    /api/v1/logs                    # 로그 검색
  Query params:
    - from: timestamp
    - to: timestamp
    - service: string
    - level: ERROR|WARN|INFO|DEBUG
    - keyword: string (regex 지원)
    - page: int
    - size: int (default: 50)
```

**Metric Controller**:
```
GET    /api/v1/metrics                 # 메트릭 조회
  Query params:
    - service: string (required)
    - metric: cpu|memory|disk|...
    - from: timestamp
    - to: timestamp
    - aggregation: avg|max|min|p95
```

**Scenario Controller** (테스트용):
```
POST   /api/v1/scenarios/{id}/trigger  # 실패 시나리오 시뮬레이션
  Path params:
    - id: scenario-1 | scenario-2 | scenario-3 | scenario-4
```

**Configuration Controller**:
```
GET    /api/v1/config/slack            # Slack 설정 조회
PUT    /api/v1/config/slack            # Slack 설정 업데이트
  Body:
    {
      "webhookUrl": "https://hooks.slack.com/...",
      "channel": "#incidents",
      "severityFilter": ["CRITICAL"],
      "enabled": true
    }
```

#### 2.2.2 Service Layer

**Incident Service**:
- 장애 이벤트 CRUD
- 페이지네이션, 필터링 로직
- 상태 변경 이력 관리

**AI Analyzer Service**:
- Claude/OpenAI API 호출
- 프롬프트 생성 (메트릭 + 로그 컨텍스트)
- 응답 파싱 및 구조화
- Fallback 로직 (Claude → OpenAI)
- 재시도 정책 (Exponential Backoff)

**구현 예시**:
```java
@Service
public class AIAnalyzerService {

    private final ClaudeClient claudeClient;
    private final OpenAIClient openAIClient;

    @Retryable(
        value = {ApiException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public AnalysisReport analyze(Incident incident) {
        String prompt = buildPrompt(incident);

        try {
            String response = claudeClient.complete(prompt);
            return parseResponse(response);
        } catch (ClaudeApiException e) {
            log.warn("Claude API failed, falling back to OpenAI", e);
            String response = openAIClient.complete(prompt);
            return parseResponse(response);
        }
    }

    private String buildPrompt(Incident incident) {
        // requirements.md FR-5 참조
        return String.format("""
            다음 시스템 장애를 분석해주세요:

            [장애 정보]
            - 발생 시각: %s
            - 영향 서비스: %s
            - 감지된 이상: %s

            [메트릭 데이터]
            %s

            [로그 샘플]
            %s

            JSON 형식으로 응답해주세요:
            {
              "root_cause": {"summary": "...", "details": "...", "confidence": 0.95},
              "impact": {...},
              "recommended_actions": [...],
              "prevention": "..."
            }
            """,
            incident.getDetectedAt(),
            incident.getServiceName(),
            getRuleDescription(incident.getRuleId()),
            getMetricsSummary(incident),
            getLogSamples(incident)
        );
    }
}
```

**Notification Service**:
- Slack Webhook API 호출
- 메시지 포맷팅 (requirements.md FR-11, FR-12 참조)
- 전송 이력 저장
- 재시도 로직

**Log Service**:
- 로그 검색 (Kafka 또는 별도 로그 저장소)
- 정규식 필터링
- 페이지네이션

**Metric Service**:
- 메트릭 데이터 조회 (MySQL 또는 시계열 DB)
- 집계 계산 (avg, max, p95 등)
- 캐싱 (Redis)

#### 2.2.3 Repository Layer

JPA Repository 사용:
```java
public interface IncidentRepository extends JpaRepository<Incident, UUID> {

    Page<Incident> findByStatus(Status status, Pageable pageable);

    Page<Incident> findBySeverityAndDetectedAtBetween(
        Severity severity,
        Instant from,
        Instant to,
        Pageable pageable
    );

    @Query("SELECT i FROM Incident i WHERE i.serviceName = :service " +
           "AND i.detectedAt > :since ORDER BY i.detectedAt DESC")
    List<Incident> findRecentByService(
        @Param("service") String service,
        @Param("since") Instant since
    );
}

public interface AnalysisReportRepository extends JpaRepository<AnalysisReport, UUID> {
    Optional<AnalysisReport> findByIncidentId(UUID incidentId);
}

public interface SlackConfigRepository extends JpaRepository<SlackConfig, Long> {
    // Singleton pattern (항상 ID=1)
}
```

### 2.3 Frontend (Web UI)

#### 2.3.1 Technology Stack
- **Framework**: React 18+ (또는 Vue 3+)
- **State Management**: React Context API (또는 Vuex)
- **HTTP Client**: Axios
- **Chart Library**: Chart.js 또는 Recharts
- **UI Components**: Material-UI (또는 Ant Design)
- **Build Tool**: Vite

#### 2.3.2 Pages

**Dashboard** (`/`):
- 실시간 장애 현황 위젯
  - OPEN 장애 개수
  - CRITICAL 장애 목록 (최근 10건)
  - 시간대별 장애 발생 추이 (차트)
- 서비스별 장애 분포 (파이 차트)

**Incident List** (`/incidents`):
- 필터: 심각도, 상태, 서비스, 기간
- 정렬: 발생시각, 심각도
- 테이블: ID, 시각, 서비스, 심각도, 상태, 액션
- 페이지네이션

**Incident Detail** (`/incidents/:id`):
- 장애 기본 정보
- AI 분석 리포트 (있는 경우)
  - Root Cause
  - Impact
  - Recommended Actions
  - Timeline
- 관련 메트릭 그래프
- 상태 변경 버튼
- "수동 분석 요청" 버튼

**Log Search** (`/logs`):
- 검색 폼 (시간, 서비스, 레벨, 키워드)
- 로그 테이블 (timestamp, level, service, message)
- 키워드 하이라이팅
- 다운로드 버튼

**Metric Viewer** (`/metrics`):
- 서비스 선택 드롭다운
- 메트릭 선택 (CPU, Memory, DB Pool 등)
- 시간 범위 선택 (1h, 6h, 24h, Custom)
- Time-series 차트
- 테이블 뷰 (옵션)

**Settings** (`/settings`):
- Slack 설정 폼
- 저장 버튼

#### 2.3.3 Component Structure
```
src/
├── components/
│   ├── Dashboard/
│   │   ├── IncidentSummary.jsx
│   │   ├── RecentIncidents.jsx
│   │   └── TrendChart.jsx
│   ├── Incident/
│   │   ├── IncidentTable.jsx
│   │   ├── IncidentDetail.jsx
│   │   ├── AnalysisReport.jsx
│   │   └── MetricChart.jsx
│   ├── Log/
│   │   ├── LogSearchForm.jsx
│   │   └── LogTable.jsx
│   └── Common/
│       ├── Header.jsx
│       ├── Sidebar.jsx
│       └── LoadingSpinner.jsx
├── services/
│   ├── incidentService.js
│   ├── logService.js
│   └── configService.js
├── contexts/
│   └── AppContext.jsx
├── App.jsx
└── main.jsx
```

### 2.4 Data Storage

#### 2.4.1 MySQL Schema

**incidents 테이블**:
```sql
CREATE TABLE incidents (
    id CHAR(36) PRIMARY KEY,
    detected_at TIMESTAMP(6) NOT NULL,
    severity ENUM('CRITICAL', 'WARNING', 'INFO') NOT NULL,
    status ENUM('OPEN', 'INVESTIGATING', 'RESOLVED') NOT NULL,
    service_name VARCHAR(100) NOT NULL,
    host_name VARCHAR(100),
    rule_id VARCHAR(50) NOT NULL,
    metric_data JSON NOT NULL,
    duplicate_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_status (status),
    INDEX idx_severity_detected (severity, detected_at),
    INDEX idx_service_detected (service_name, detected_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**analysis_reports 테이블**:
```sql
CREATE TABLE analysis_reports (
    id CHAR(36) PRIMARY KEY,
    incident_id CHAR(36) NOT NULL UNIQUE,
    analyzed_at TIMESTAMP(6) NOT NULL,
    root_cause JSON NOT NULL,
    impact JSON NOT NULL,
    recommended_actions JSON NOT NULL,
    timeline JSON,
    prevention TEXT,
    ai_provider ENUM('CLAUDE', 'OPENAI') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (incident_id) REFERENCES incidents(id) ON DELETE CASCADE,
    INDEX idx_incident (incident_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**slack_configs 테이블** (Singleton):
```sql
CREATE TABLE slack_configs (
    id BIGINT PRIMARY KEY DEFAULT 1,
    webhook_url VARCHAR(500) NOT NULL,
    channel VARCHAR(100),
    severity_filter JSON, -- ["CRITICAL", "WARNING"]
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    CHECK (id = 1) -- Singleton constraint
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**notification_logs 테이블**:
```sql
CREATE TABLE notification_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    incident_id CHAR(36) NOT NULL,
    type ENUM('INCIDENT_DETECTED', 'ANALYSIS_COMPLETED') NOT NULL,
    sent_at TIMESTAMP(6) NOT NULL,
    success BOOLEAN NOT NULL,
    error_message TEXT,

    FOREIGN KEY (incident_id) REFERENCES incidents(id) ON DELETE CASCADE,
    INDEX idx_incident (incident_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Flyway 마이그레이션**:
```
src/main/resources/db/migration/
├── V1__create_incidents_table.sql
├── V2__create_analysis_reports_table.sql
├── V3__create_slack_configs_table.sql
└── V4__create_notification_logs_table.sql
```

#### 2.4.2 Redis Schema

**Cache Keys**:
```
# 분석 리포트 캐싱
analysis:report:{incident_id} → JSON (TTL: 1시간)

# 중복 이벤트 체크
incident:dedup:{service}:{metric} → Sorted Set (TTL: 5분)

# 메트릭 집계 캐싱
metrics:{service}:{metric}:{from}:{to}:{agg} → JSON (TTL: 5분)

# Slack 설정 캐싱
config:slack → JSON (TTL: 10분, 업데이트 시 무효화)
```

**Example**:
```redis
# 중복 체크
ZADD incident:dedup:payment-service:db.pool 1708000000 "INC-001"
ZRANGEBYSCORE incident:dedup:payment-service:db.pool 1707999700 +inf
# → 5분 내 동일 이벤트 존재 시 중복 처리

# 분석 리포트 캐싱
SET analysis:report:INC-001 '{"root_cause": {...}}' EX 3600
GET analysis:report:INC-001
```

### 2.5 External Integrations

#### 2.5.1 Claude API Integration

**Client 구현**:
```java
@Component
public class ClaudeClient {

    @Value("${claude.api.key}")
    private String apiKey;

    @Value("${claude.api.model:claude-3-5-sonnet-20240620}")
    private String model;

    private final RestTemplate restTemplate;

    public String complete(String prompt) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("x-api-key", apiKey);
        headers.set("anthropic-version", "2023-06-01");
        headers.setContentType(MediaType.APPLICATION_JSON);

        Map<String, Object> request = Map.of(
            "model", model,
            "max_tokens", 4096,
            "temperature", 0.3,
            "messages", List.of(
                Map.of("role", "user", "content", prompt)
            )
        );

        HttpEntity<Map<String, Object>> entity =
            new HttpEntity<>(request, headers);

        ResponseEntity<Map> response = restTemplate.postForEntity(
            "https://api.anthropic.com/v1/messages",
            entity,
            Map.class
        );

        return extractContent(response.getBody());
    }
}
```

#### 2.5.2 Slack Webhook Integration

**Client 구현**:
```java
@Component
public class SlackClient {

    private final RestTemplate restTemplate;
    private final SlackConfigRepository configRepository;

    public void sendIncidentAlert(Incident incident) {
        SlackConfig config = configRepository.findById(1L)
            .orElseThrow();

        if (!config.isEnabled()) {
            return;
        }

        String message = formatIncidentMessage(incident);

        Map<String, Object> payload = Map.of(
            "channel", config.getChannel(),
            "text", message,
            "username", "Incident Bot",
            "icon_emoji", ":rotating_light:"
        );

        restTemplate.postForEntity(
            config.getWebhookUrl(),
            payload,
            String.class
        );
    }

    private String formatIncidentMessage(Incident incident) {
        // requirements.md FR-11 포맷 참조
        return String.format("""
            🚨 CRITICAL Incident Detected

            Incident ID: %s
            Service: %s
            ...
            """, incident.getId(), incident.getServiceName());
    }
}
```

## 3. Infrastructure

### 3.1 Docker Compose Setup

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"

  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: incident_analysis
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  incident-system:
    build: ./backend
    depends_on:
      - kafka
      - mysql
      - redis
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/incident_analysis
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      CLAUDE_API_KEY: ${CLAUDE_API_KEY}
      OPENAI_API_KEY: ${OPENAI_API_KEY}

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      VITE_API_BASE_URL: http://localhost:8080

volumes:
  mysql-data:
```

### 3.2 Application Configuration

**application.yml**:
```yaml
spring:
  application:
    name: incident-analysis-system

  datasource:
    url: jdbc:mysql://localhost:3306/incident_analysis
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQL8Dialect

  flyway:
    enabled: true
    locations: classpath:db/migration

  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: incident-detection-group
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    listener:
      ack-mode: manual

  data:
    redis:
      host: localhost
      port: 6379

claude:
  api:
    key: ${CLAUDE_API_KEY}
    model: claude-3-5-sonnet-20240620
    base-url: https://api.anthropic.com/v1

openai:
  api:
    key: ${OPENAI_API_KEY}
    model: gpt-4-turbo-preview
    base-url: https://api.openai.com/v1

logging:
  level:
    root: INFO
    com.incident.analysis: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

## 4. Data Flow

### 4.1 Incident Detection Flow

```
1. Kafka 메시지 수신
   ↓
2. Rule Engine 평가
   ↓
3. [규칙 위반?] → No → 무시
   ↓ Yes
4. Redis 중복 체크
   ↓
5. [중복?] → Yes → 카운트만 증가
   ↓ No
6. Incident 생성 및 MySQL 저장
   ↓
7. [CRITICAL?] → Yes → AI 분석 트리거
   ↓              ↓
8. Slack 알림    AI Analyzer 비동기 실행
```

### 4.2 AI Analysis Flow

```
1. AI Analyzer 트리거
   ↓
2. 컨텍스트 수집
   - 메트릭 데이터 (15분 전후)
   - 로그 샘플 (1시간)
   - 배포 이력
   ↓
3. 프롬프트 생성
   ↓
4. Claude API 호출
   ↓
5. [성공?] → No → OpenAI Fallback
   ↓ Yes           ↓
6. 응답 파싱 ← ← ← ┘
   ↓
7. AnalysisReport 생성
   ↓
8. MySQL 저장 + Redis 캐싱
   ↓
9. Slack 분석 완료 알림
```

### 4.3 API Request Flow

```
User → Web UI → REST API → Service Layer → Repository → MySQL/Redis
                    ↓
                 Response
```

## 5. Deployment Architecture (Future)

초기 버전은 Docker Compose 기반 단일 서버 배포이지만, 향후 확장 고려사항:

```
┌─────────────────────────────────────────────────────┐
│                  Load Balancer                       │
└──────────────┬──────────────────────────────────────┘
               │
       ┌───────┴───────┐
       ▼               ▼
┌─────────────┐  ┌─────────────┐
│ Spring Boot │  │ Spring Boot │  (Horizontal Scaling)
│ Instance 1  │  │ Instance 2  │
└──────┬──────┘  └──────┬──────┘
       │                │
       └────────┬───────┘
                ▼
       ┌─────────────────┐
       │  MySQL Cluster  │  (Master-Slave Replication)
       └─────────────────┘

       ┌─────────────────┐
       │  Redis Cluster  │  (Sentinel or Cluster Mode)
       └─────────────────┘

       ┌─────────────────┐
       │ Kafka Cluster   │  (3+ Brokers)
       └─────────────────┘
```

## 6. Security Architecture

### 6.1 Authentication & Authorization
- JWT 토큰 기반 인증
- Token 유효기간: 24시간
- Refresh Token: 7일

### 6.2 Secrets Management
- 환경변수로 관리: `CLAUDE_API_KEY`, `OPENAI_API_KEY`, `SLACK_WEBHOOK_URL`
- Production: AWS Secrets Manager 또는 HashiCorp Vault 사용
- 절대 코드에 하드코딩 금지

### 6.3 API Security
- HTTPS only (Production)
- Rate Limiting: 100 req/min per user
- SQL Injection 방지: Prepared Statement
- XSS 방지: 입력 Sanitization

## 7. Monitoring & Observability

### 7.1 Metrics
- Spring Boot Actuator + Micrometer
- Prometheus 형식으로 export
- 주요 메트릭:
  - `incident.detected.total` (counter)
  - `ai.analysis.duration` (histogram)
  - `kafka.consumer.lag` (gauge)
  - `http.request.duration` (histogram)

### 7.2 Logging
- Logback + JSON format
- 로그 레벨: INFO (기본), DEBUG (개발)
- 구조화된 로그:
  ```json
  {
    "timestamp": "2026-02-14T14:32:15.123Z",
    "level": "INFO",
    "logger": "com.incident.analysis.service.AIAnalyzerService",
    "message": "AI analysis completed",
    "incident_id": "INC-001",
    "duration_ms": 1234
  }
  ```

### 7.3 Health Checks
```
GET /actuator/health

Response:
{
  "status": "UP",
  "components": {
    "db": {"status": "UP"},
    "redis": {"status": "UP"},
    "kafka": {"status": "UP"},
    "diskSpace": {"status": "UP"}
  }
}
```

## 8. Technology Stack Summary

| Layer | Technology | Version |
|-------|------------|---------|
| Backend | Spring Boot | 3.2.x |
| Language | Java | 17+ |
| Build Tool | Gradle | 8.x |
| Database | MySQL | 8.0 |
| Migration | Flyway | 10.x |
| Cache | Redis | 7.x |
| Messaging | Apache Kafka | 3.6.x |
| ORM | Spring Data JPA | (with Spring Boot) |
| AI API | Claude API | v1 |
| AI API (Fallback) | OpenAI API | v1 |
| Alerting | Slack Webhook | - |
| Frontend | React | 18+ |
| State | React Context | - |
| HTTP Client | Axios | 1.6.x |
| Charts | Chart.js | 4.x |
| UI Library | Material-UI | 5.x |
| Build (FE) | Vite | 5.x |
| Container | Docker | 24.x |
| Orchestration | Docker Compose | v3.8 |
