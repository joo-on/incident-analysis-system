# API Specification

## Base URL

```
Local: http://localhost:8080/api/v1
Production: https://incident-system.example.com/api/v1
```

## Authentication

모든 API 요청은 JWT 토큰을 필요로 합니다 (v1.0에서는 간소화를 위해 선택 사항).

```
Authorization: Bearer {jwt_token}
```

## Common Response Format

### Success Response
```json
{
  "success": true,
  "data": { ... },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

### Error Response
```json
{
  "success": false,
  "error": {
    "code": "INCIDENT_NOT_FOUND",
    "message": "Incident with ID 'abc-123' not found",
    "details": { ... }
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

### HTTP Status Codes
- `200 OK` - 요청 성공
- `201 Created` - 리소스 생성 성공
- `400 Bad Request` - 잘못된 요청 파라미터
- `404 Not Found` - 리소스를 찾을 수 없음
- `500 Internal Server Error` - 서버 내부 오류
- `503 Service Unavailable` - AI API 장애 등

---

## 1. Incident API

### 1.1 List Incidents

**Endpoint**: `GET /incidents`

**Description**: 장애 목록을 조회합니다. 페이징 및 필터링을 지원합니다.

**Query Parameters**:
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| page | integer | No | 0 | 페이지 번호 (0-based) |
| size | integer | No | 20 | 페이지 크기 (max: 100) |
| severity | string | No | - | 심각도 필터 (CRITICAL, WARNING, INFO) |
| status | string | No | - | 상태 필터 (OPEN, INVESTIGATING, RESOLVED) |
| service | string | No | - | 서비스 이름 필터 |
| from | string (ISO 8601) | No | - | 시작 시각 (UTC) |
| to | string (ISO 8601) | No | - | 종료 시각 (UTC) |
| sort | string | No | detectedAt,desc | 정렬 기준 (field,direction) |

**Example Request**:
```http
GET /api/v1/incidents?severity=CRITICAL&status=OPEN&page=0&size=10
```

**Example Response**:
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "detectedAt": "2026-02-14T14:30:00.000Z",
        "severity": "CRITICAL",
        "status": "OPEN",
        "serviceName": "payment-service",
        "hostName": "payment-prod-01",
        "ruleId": "db.connection.pool.exhausted",
        "duplicateCount": 3,
        "hasAnalysisReport": true
      }
    ],
    "page": {
      "number": 0,
      "size": 10,
      "totalElements": 42,
      "totalPages": 5
    }
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

---

### 1.2 Get Incident Detail

**Endpoint**: `GET /incidents/{id}`

**Description**: 특정 장애의 상세 정보를 조회합니다.

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | UUID | Yes | Incident ID |

**Example Request**:
```http
GET /api/v1/incidents/550e8400-e29b-41d4-a716-446655440000
```

**Example Response**:
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "detectedAt": "2026-02-14T14:30:00.000Z",
    "severity": "CRITICAL",
    "status": "OPEN",
    "serviceName": "payment-service",
    "hostName": "payment-prod-01",
    "ruleId": "db.connection.pool.exhausted",
    "ruleDescription": "Database connection pool usage > 90% for 2 minutes",
    "metricData": {
      "metric": "db.connection.pool.usage",
      "value": 98.5,
      "threshold": 90.0,
      "duration": "PT5M",
      "activeConnections": 49,
      "maxConnections": 50,
      "pendingRequests": 127
    },
    "duplicateCount": 3,
    "createdAt": "2026-02-14T14:30:00.123Z",
    "updatedAt": "2026-02-14T14:32:00.456Z"
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

**Error Response** (404):
```json
{
  "success": false,
  "error": {
    "code": "INCIDENT_NOT_FOUND",
    "message": "Incident with ID '550e8400-e29b-41d4-a716-446655440000' not found"
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

---

### 1.3 Update Incident Status

**Endpoint**: `PATCH /incidents/{id}/status`

**Description**: 장애의 상태를 변경합니다.

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | UUID | Yes | Incident ID |

**Request Body**:
```json
{
  "status": "INVESTIGATING",
  "comment": "팀에서 조사 시작"
}
```

**Body Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| status | string | Yes | 새 상태 (INVESTIGATING, RESOLVED) |
| comment | string | No | 상태 변경 사유 |

**Example Response**:
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "INVESTIGATING",
    "updatedAt": "2026-02-14T14:35:00.789Z"
  },
  "timestamp": "2026-02-14T14:35:00.789Z"
}
```

**Validation Errors** (400):
```json
{
  "success": false,
  "error": {
    "code": "INVALID_STATUS_TRANSITION",
    "message": "Cannot transition from RESOLVED to INVESTIGATING",
    "details": {
      "currentStatus": "RESOLVED",
      "requestedStatus": "INVESTIGATING",
      "allowedTransitions": []
    }
  },
  "timestamp": "2026-02-14T14:35:00.789Z"
}
```

---

### 1.4 Get Analysis Report

**Endpoint**: `GET /incidents/{id}/report`

**Description**: 장애의 AI 분석 리포트를 조회합니다.

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | UUID | Yes | Incident ID |

**Example Request**:
```http
GET /api/v1/incidents/550e8400-e29b-41d4-a716-446655440000/report
```

**Example Response**:
```json
{
  "success": true,
  "data": {
    "id": "650e8400-e29b-41d4-a716-446655440001",
    "incidentId": "550e8400-e29b-41d4-a716-446655440000",
    "analyzedAt": "2026-02-14T14:31:45.678Z",
    "aiProvider": "CLAUDE",
    "rootCause": {
      "summary": "Connection leak in UserService.getUserDetails()",
      "details": "The method fails to close database connections in the exception handler path, causing gradual pool exhaustion over time.",
      "confidence": 0.95
    },
    "impact": {
      "affectedServices": ["payment-service", "order-service"],
      "estimatedUsers": 1500,
      "severity": "CRITICAL",
      "businessImpact": "Payment processing blocked, potential revenue loss ~$50K/hour"
    },
    "recommendedActions": [
      {
        "priority": 1,
        "action": "Restart payment-service immediately to release connections",
        "expectedResult": "Immediate recovery of service availability",
        "estimatedDuration": "PT2M"
      },
      {
        "priority": 2,
        "action": "Deploy hotfix PR#1234 to fix connection leak",
        "expectedResult": "Prevent recurrence of this issue",
        "estimatedDuration": "PT1H"
      },
      {
        "priority": 3,
        "action": "Add connection pool monitoring alert (threshold: 80%)",
        "expectedResult": "Early warning before exhaustion",
        "estimatedDuration": "PT30M"
      }
    ],
    "timeline": [
      {
        "time": "2026-02-14T14:00:00.000Z",
        "event": "Normal operation, pool usage ~30%"
      },
      {
        "time": "2026-02-14T14:15:00.000Z",
        "event": "Increased user traffic, pool usage rising to 60%"
      },
      {
        "time": "2026-02-14T14:25:00.000Z",
        "event": "Connection leak accumulation, pool usage > 90%"
      },
      {
        "time": "2026-02-14T14:30:00.000Z",
        "event": "Pool exhaustion detected, incident triggered"
      }
    ],
    "prevention": "Implement try-with-resources for all database operations and add automated connection leak detection in CI/CD pipeline.",
    "createdAt": "2026-02-14T14:31:45.678Z"
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

**Error Response** (404):
```json
{
  "success": false,
  "error": {
    "code": "REPORT_NOT_FOUND",
    "message": "No analysis report found for incident '550e8400-e29b-41d4-a716-446655440000'"
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

---

### 1.5 Trigger Manual Analysis

**Endpoint**: `POST /incidents/{id}/analyze`

**Description**: 장애에 대한 AI 분석을 수동으로 트리거합니다.

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | UUID | Yes | Incident ID |

**Request Body**: (empty)

**Example Response**:
```json
{
  "success": true,
  "data": {
    "incidentId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "ANALYZING",
    "message": "AI analysis started. You will be notified when completed.",
    "estimatedCompletionTime": "2026-02-14T14:34:00.000Z"
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

**Error Response** (400 - 이미 분석 중):
```json
{
  "success": false,
  "error": {
    "code": "ANALYSIS_IN_PROGRESS",
    "message": "Analysis is already in progress for this incident"
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

---

## 2. Log API

### 2.1 Search Logs

**Endpoint**: `GET /logs`

**Description**: 시스템 로그를 검색합니다.

**Query Parameters**:
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| from | string (ISO 8601) | Yes | - | 시작 시각 (UTC) |
| to | string (ISO 8601) | Yes | - | 종료 시각 (UTC) |
| service | string | No | - | 서비스 이름 필터 |
| level | string | No | - | 로그 레벨 (ERROR, WARN, INFO, DEBUG) |
| keyword | string | No | - | 검색 키워드 (정규식 지원) |
| host | string | No | - | 호스트 이름 필터 |
| page | integer | No | 0 | 페이지 번호 |
| size | integer | No | 50 | 페이지 크기 (max: 500) |

**Example Request**:
```http
GET /api/v1/logs?from=2026-02-14T14:00:00Z&to=2026-02-14T15:00:00Z&service=payment-service&level=ERROR&keyword=SQLException&page=0&size=50
```

**Example Response**:
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "timestamp": "2026-02-14T14:30:12.345Z",
        "level": "ERROR",
        "service": "payment-service",
        "host": "payment-prod-01",
        "logger": "com.payment.service.UserService",
        "message": "Failed to execute query: java.sql.SQLException: Connection pool exhausted",
        "stackTrace": "java.sql.SQLException: Connection pool exhausted\n  at com.zaxxer.hikari.pool.HikariPool.getConnection...",
        "context": {
          "userId": "12345",
          "requestId": "req-abc-123",
          "threadName": "http-nio-8080-exec-15"
        }
      }
    ],
    "page": {
      "number": 0,
      "size": 50,
      "totalElements": 127,
      "totalPages": 3
    }
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

**Validation Errors** (400):
```json
{
  "success": false,
  "error": {
    "code": "INVALID_TIME_RANGE",
    "message": "Time range cannot exceed 24 hours",
    "details": {
      "from": "2026-02-14T00:00:00Z",
      "to": "2026-02-16T00:00:00Z",
      "maxRangeHours": 24
    }
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

---

## 3. Metric API

### 3.1 Get Metrics

**Endpoint**: `GET /metrics`

**Description**: 서비스의 메트릭 데이터를 조회합니다.

**Query Parameters**:
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| service | string | Yes | - | 서비스 이름 |
| metric | string | Yes | - | 메트릭 이름 (cpu, memory, disk, db.pool, etc.) |
| from | string (ISO 8601) | Yes | - | 시작 시각 (UTC) |
| to | string (ISO 8601) | Yes | - | 종료 시각 (UTC) |
| aggregation | string | No | avg | 집계 방식 (avg, max, min, p50, p95, p99) |
| interval | string | No | 1m | 집계 간격 (1m, 5m, 1h) |

**Available Metrics**:
- `cpu` - CPU 사용률 (%)
- `memory` - Memory 사용률 (%)
- `disk` - Disk 사용률 (%)
- `db.pool` - DB Connection Pool 사용률 (%)
- `kafka.lag` - Kafka Consumer Lag (messages)
- `http.request.rate` - HTTP Request Rate (req/sec)
- `http.error.rate` - HTTP Error Rate (%)
- `http.response.time` - HTTP Response Time (ms)

**Example Request**:
```http
GET /api/v1/metrics?service=payment-service&metric=db.pool&from=2026-02-14T14:00:00Z&to=2026-02-14T15:00:00Z&aggregation=max&interval=1m
```

**Example Response**:
```json
{
  "success": true,
  "data": {
    "service": "payment-service",
    "metric": "db.pool",
    "aggregation": "max",
    "interval": "1m",
    "unit": "%",
    "dataPoints": [
      {
        "timestamp": "2026-02-14T14:00:00.000Z",
        "value": 32.5
      },
      {
        "timestamp": "2026-02-14T14:01:00.000Z",
        "value": 35.2
      },
      {
        "timestamp": "2026-02-14T14:02:00.000Z",
        "value": 38.1
      },
      {
        "timestamp": "2026-02-14T14:30:00.000Z",
        "value": 98.5
      }
    ],
    "summary": {
      "avg": 52.3,
      "max": 98.5,
      "min": 30.1,
      "p95": 87.2
    }
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

---

## 4. Scenario API (Testing)

### 4.1 Trigger Scenario

**Endpoint**: `POST /scenarios/{scenarioId}/trigger`

**Description**: 실패 시나리오를 시뮬레이션합니다 (테스트 목적).

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| scenarioId | string | Yes | scenario-1, scenario-2, scenario-3, scenario-4 |

**Scenario IDs**:
- `scenario-1` - Database Connection Pool Exhaustion
- `scenario-2` - Kafka Consumer Lag Spike
- `scenario-3` - Memory Leak Detection
- `scenario-4` - Cascading Service Failure

**Request Body**:
```json
{
  "duration": "PT5M",
  "intensity": "HIGH"
}
```

**Body Parameters**:
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| duration | string (ISO 8601 Duration) | No | PT5M | 시나리오 지속 시간 |
| intensity | string | No | MEDIUM | 강도 (LOW, MEDIUM, HIGH) |

**Example Request**:
```http
POST /api/v1/scenarios/scenario-1/trigger
Content-Type: application/json

{
  "duration": "PT5M",
  "intensity": "HIGH"
}
```

**Example Response**:
```json
{
  "success": true,
  "data": {
    "scenarioId": "scenario-1",
    "scenarioName": "Database Connection Pool Exhaustion",
    "status": "RUNNING",
    "startedAt": "2026-02-14T14:32:15.123Z",
    "expectedEndAt": "2026-02-14T14:37:15.123Z",
    "message": "Scenario simulation started. Monitor incidents for detection.",
    "monitoringUrl": "/api/v1/incidents?service=payment-service"
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

**Error Response** (400):
```json
{
  "success": false,
  "error": {
    "code": "SCENARIO_NOT_FOUND",
    "message": "Scenario 'scenario-5' does not exist",
    "details": {
      "availableScenarios": ["scenario-1", "scenario-2", "scenario-3", "scenario-4"]
    }
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

---

## 5. Configuration API

### 5.1 Get Slack Configuration

**Endpoint**: `GET /config/slack`

**Description**: Slack 알림 설정을 조회합니다.

**Example Response**:
```json
{
  "success": true,
  "data": {
    "webhookUrl": "https://hooks.slack.com/services/EXAMPLE/EXAMPLE/ExampleWebhookUrlNotReal",
    "channel": "#incidents",
    "severityFilter": ["CRITICAL", "WARNING"],
    "enabled": true,
    "updatedAt": "2026-02-14T10:00:00.000Z"
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

---

### 5.2 Update Slack Configuration

**Endpoint**: `PUT /config/slack`

**Description**: Slack 알림 설정을 업데이트합니다.

**Request Body**:
```json
{
  "webhookUrl": "https://hooks.slack.com/services/EXAMPLE/EXAMPLE/ExampleWebhookUrlNotReal",
  "channel": "#incidents-critical",
  "severityFilter": ["CRITICAL"],
  "enabled": true
}
```

**Body Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| webhookUrl | string | Yes | Slack Webhook URL |
| channel | string | No | 알림 받을 채널 |
| severityFilter | array[string] | No | 알림받을 심각도 목록 |
| enabled | boolean | No | 알림 활성화 여부 |

**Example Response**:
```json
{
  "success": true,
  "data": {
    "webhookUrl": "https://hooks.slack.com/services/EXAMPLE/EXAMPLE/ExampleWebhookUrlNotReal",
    "channel": "#incidents-critical",
    "severityFilter": ["CRITICAL"],
    "enabled": true,
    "updatedAt": "2026-02-14T14:35:00.789Z"
  },
  "timestamp": "2026-02-14T14:35:00.789Z"
}
```

**Validation Errors** (400):
```json
{
  "success": false,
  "error": {
    "code": "INVALID_WEBHOOK_URL",
    "message": "Webhook URL must start with 'https://hooks.slack.com/'"
  },
  "timestamp": "2026-02-14T14:35:00.789Z"
}
```

---

## 6. Health Check API

### 6.1 Health Check

**Endpoint**: `GET /actuator/health`

**Description**: 시스템 헬스 상태를 조회합니다.

**Example Response**:
```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MySQL",
        "validationQuery": "isValid()"
      }
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "7.2.3"
      }
    },
    "kafka": {
      "status": "UP",
      "details": {
        "clusterId": "kafka-cluster-01"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 268435456000,
        "free": 107374182400,
        "threshold": 10485760
      }
    }
  }
}
```

---

## 7. Error Codes Reference

| Code | HTTP Status | Description |
|------|-------------|-------------|
| INCIDENT_NOT_FOUND | 404 | 요청한 Incident를 찾을 수 없음 |
| REPORT_NOT_FOUND | 404 | 분석 리포트가 존재하지 않음 |
| SCENARIO_NOT_FOUND | 404 | 요청한 시나리오가 존재하지 않음 |
| INVALID_STATUS_TRANSITION | 400 | 허용되지 않는 상태 전환 |
| INVALID_TIME_RANGE | 400 | 잘못된 시간 범위 (24시간 초과 등) |
| INVALID_WEBHOOK_URL | 400 | 잘못된 Slack Webhook URL |
| ANALYSIS_IN_PROGRESS | 400 | 이미 분석이 진행 중 |
| AI_API_ERROR | 503 | AI API 호출 실패 |
| DATABASE_ERROR | 500 | 데이터베이스 오류 |
| INTERNAL_SERVER_ERROR | 500 | 예상하지 못한 서버 오류 |

---

## 8. Rate Limiting

- **Limit**: 100 requests/minute per user
- **Header**:
  ```
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 87
  X-RateLimit-Reset: 1708000000
  ```

**Rate Limit Exceeded Response** (429):
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 45 seconds.",
    "details": {
      "limit": 100,
      "resetAt": "2026-02-14T14:33:00.000Z"
    }
  },
  "timestamp": "2026-02-14T14:32:15.123Z"
}
```

---

## 9. Swagger/OpenAPI Documentation

API 문서는 Swagger UI를 통해 인터랙티브하게 제공됩니다:

```
http://localhost:8080/swagger-ui.html
```

OpenAPI 3.0 스펙:
```
http://localhost:8080/v3/api-docs
```
