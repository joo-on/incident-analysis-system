# Evaluation Criteria

이 문서는 모든 AI 도구 구현을 평가하는 기준을 정의합니다. 공정한 비교를 위해 모든 구현은 동일한 기준으로 평가됩니다.

## 평가 프로세스

```
1. 구현 완료 확인
2. 자동화된 테스트 실행
3. 정량적 메트릭 수집
4. 정성적 코드 리뷰
5. 시나리오 검증
6. 평가 리포트 작성
```

---

## 1. 기능 완성도 (40점)

### 1.1 Core Features (25점)

#### 실시간 장애 감지 (7점)
- [ ] **[3점]** 6개 감지 규칙 모두 구현 (CPU, Memory, DB Pool, Kafka Lag, HTTP Error, Response Time)
- [ ] **[2점]** Kafka Consumer 정상 동작 (3개 파티션 병렬 처리)
- [ ] **[1점]** 중복 이벤트 필터링 (Redis 기반, 5분 윈도우)
- [ ] **[1점]** Incident 엔티티 MySQL 저장

#### AI 기반 근본 원인 분석 (10점)
- [ ] **[3점]** Claude API 연동 및 정상 호출
- [ ] **[2점]** OpenAI Fallback 로직 구현
- [ ] **[2점]** 프롬프트 컨텍스트 구성 (메트릭 + 로그 + 배포 이력)
- [ ] **[2점]** AI 응답 파싱 및 AnalysisReport 엔티티 저장
- [ ] **[1점]** 재시도 로직 (Exponential Backoff, 최대 3회)

#### Slack 알림 (4점)
- [ ] **[2점]** CRITICAL 장애 즉시 알림 (중복 제거 포함)
- [ ] **[1점]** AI 분석 완료 알림
- [ ] **[1점]** Slack 설정 API (조회/업데이트)

#### Web UI (4점)
- [ ] **[1점]** Dashboard (실시간 장애 현황)
- [ ] **[1점]** Incident List & Detail 페이지
- [ ] **[1점]** Log Search 기능
- [ ] **[1점]** Metric Viewer (시계열 차트)

### 1.2 시나리오 검증 (15점)

각 시나리오당 3.75점 (총 4개)

**Scenario 1: Database Connection Pool Exhaustion**
- [ ] **[1.25점]** 시나리오 트리거 API 구현
- [ ] **[1.25점]** 60초 이내 장애 감지
- [ ] **[1.25점]** AI 분석 완료 및 정확한 근본 원인 식별

**Scenario 2: Kafka Consumer Lag Spike**
- [ ] **[1.25점]** 시나리오 트리거 API 구현
- [ ] **[1.25점]** 60초 이내 장애 감지
- [ ] **[1.25점]** AI 분석 완료 및 정확한 근본 원인 식별

**Scenario 3: Memory Leak Detection**
- [ ] **[1.25점]** 시나리오 트리거 API 구현
- [ ] **[1.25점]** 60초 이내 장애 감지
- [ ] **[1.25점]** AI 분석 완료 및 정확한 근본 원인 식별

**Scenario 4: Cascading Service Failure**
- [ ] **[1.25점]** 시나리오 트리거 API 구현
- [ ] **[1.25점]** 60초 이내 장애 감지
- [ ] **[1.25점]** AI 분석 완료 및 정확한 근본 원인 식별

---

## 2. 코드 품질 (25점)

### 2.1 테스트 커버리지 (8점)
- [ ] **[3점]** 단위 테스트 커버리지 > 70%
- [ ] **[3점]** 통합 테스트로 4개 시나리오 자동 검증
- [ ] **[2점]** 주요 서비스/컨트롤러 모두 테스트 존재

**측정 방법**:
```bash
./gradlew test jacocoTestReport
# 리포트: build/reports/jacoco/test/html/index.html
```

### 2.2 코드 스타일 & 가독성 (7점)
- [ ] **[2점]** 일관된 네이밍 컨벤션 (camelCase, PascalCase)
- [ ] **[2점]** 적절한 메서드 분리 (Single Responsibility)
- [ ] **[2점]** 의미 있는 변수/메서드명 (축약어 최소화)
- [ ] **[1점]** 매직 넘버 없음 (상수로 정의)

**평가 기준**:
- 클래스당 평균 메서드 수 < 15
- 메서드당 평균 라인 수 < 30
- Cyclomatic Complexity < 10 (메서드당)

### 2.3 아키텍처 일관성 (5점)
- [ ] **[2점]** Layered Architecture 준수 (Controller → Service → Repository)
- [ ] **[1점]** DTO와 Entity 분리
- [ ] **[1점]** 순환 의존성 없음
- [ ] **[1점]** 적절한 패키지 구조

**패키지 구조 예시**:
```
com.incident.analysis
├── controller
├── service
├── repository
├── entity
├── dto
├── config
└── exception
```

### 2.4 에러 핸들링 (5점)
- [ ] **[2점]** Global Exception Handler 구현 (`@ControllerAdvice`)
- [ ] **[2점]** 커스텀 예외 클래스 정의 및 사용
- [ ] **[1점]** 적절한 HTTP 상태 코드 반환 (400, 404, 500 등)

**필수 예외**:
- `IncidentNotFoundException`
- `AnalysisReportNotFoundException`
- `AiApiException`
- `InvalidStatusTransitionException`

---

## 3. 비기능 요구사항 (20점)

### 3.1 성능 (7점)
- [ ] **[2점]** 실시간 이벤트 처리 지연 < 1초 (P95)
- [ ] **[2점]** AI 분석 완료 시간 < 2분 (P95)
- [ ] **[2점]** API 응답 시간 < 500ms (P95, 조회 API)
- [ ] **[1점]** Kafka 메시지 처리량 > 1,000 msg/sec (테스트 환경 기준)

**측정 방법**:
- JMeter 또는 Gatling으로 부하 테스트
- Spring Boot Actuator metrics 확인

### 3.2 보안 (6점)
- [ ] **[2점]** AI API Key 환경변수 관리 (코드에 하드코딩 없음)
- [ ] **[2점]** SQL Injection 방지 (Prepared Statement/JPA 사용)
- [ ] **[1점]** Slack Webhook URL 암호화 저장 (AES-256)
- [ ] **[1점]** 입력 값 검증 (Bean Validation 사용)

**보안 체크리스트**:
```bash
# 하드코딩된 시크릿 스캔
grep -r "sk-" src/  # OpenAI API Key 패턴
grep -r "claude" src/ | grep "api" | grep "key"

# SQL Injection 취약점
grep -r "createNativeQuery" src/ # Native Query 사용 확인
```

### 3.3 운영 준비도 (7점)
- [ ] **[2점]** Docker Compose로 로컬 실행 가능
- [ ] **[2점]** Health Check 엔드포인트 구현 (DB, Redis, Kafka 상태 확인)
- [ ] **[2점]** 구조화된 로깅 (JSON format, 주요 이벤트 로깅)
- [ ] **[1점]** Prometheus 메트릭 export (Micrometer)

**필수 로그**:
```java
// 장애 감지
log.info("Incident detected", Map.of(
    "incidentId", incident.getId(),
    "severity", incident.getSeverity(),
    "service", incident.getServiceName()
));

// AI 분석 시작/완료
log.info("AI analysis started", Map.of("incidentId", id));
log.info("AI analysis completed", Map.of(
    "incidentId", id,
    "duration", duration,
    "provider", "CLAUDE"
));

// Slack 알림
log.info("Slack notification sent", Map.of(
    "incidentId", id,
    "type", "INCIDENT_DETECTED"
));
```

---

## 4. 문서화 (10점)

### 4.1 코드 문서화 (4점)
- [ ] **[2점]** API 문서 (Swagger/OpenAPI)
  - 모든 엔드포인트 문서화
  - Request/Response 예시 포함
- [ ] **[1점]** JavaDoc (주요 클래스/메서드)
- [ ] **[1점]** README.md (실행 방법, 환경 변수)

### 4.2 아키텍처 문서화 (3점)
- [ ] **[2점]** 시스템 아키텍처 다이어그램 (ASCII 또는 이미지)
- [ ] **[1점]** 주요 플로우 설명 (장애 감지 → AI 분석 → 알림)

### 4.3 운영 문서화 (3점)
- [ ] **[1점]** 트러블슈팅 가이드 (자주 발생하는 오류 및 해결 방법)
- [ ] **[1점]** 환경 변수 리스트 (`.env.example` 제공)
- [ ] **[1점]** 로컬 개발 환경 설정 가이드

---

## 5. 구현 효율성 (5점)

### 5.1 AI 협업 효율성 (3점)
- [ ] **[1점]** 총 대화 턴 수 < 50회
- [ ] **[1점]** 소요 시간 < 8시간 (순수 구현 시간, 학습 시간 제외)
- [ ] **[1점]** AI 제안 수용률 > 70% (재작업 최소화)

**측정 방법**:
- `prompts/{implementation}.md`에 기록된 대화 턴 수
- 타임스탬프 기반 소요 시간 계산
- AI 제안을 그대로 수용한 비율

### 5.2 코드 재사용성 (2점)
- [ ] **[1점]** 중복 코드 < 5% (SonarQube Duplicated Lines)
- [ ] **[1점]** 적절한 유틸리티 클래스/메서드 사용

---

## 6. 보너스 점수 (최대 +10점)

### 6.1 추가 기능 구현
- **[+2점]** 다국어 지원 (한국어, 영어)
- **[+2점]** 실시간 WebSocket 알림 (Web UI)
- **[+2점]** 커스텀 감지 규칙 생성 UI
- **[+2점]** 장애 통계 대시보드 (주간/월간 리포트)
- **[+2점]** CI/CD 파이프라인 구성 (GitHub Actions)

### 6.2 성능 최적화
- **[+3점]** 이벤트 처리 지연 < 500ms (P95)
- **[+2점]** AI 분석 완료 시간 < 1분 (P95)

---

## 평가 점수표

| 카테고리 | 배점 | 최소 합격 점수 |
|----------|------|----------------|
| 1. 기능 완성도 | 40점 | 32점 (80%) |
| 2. 코드 품질 | 25점 | 18점 (72%) |
| 3. 비기능 요구사항 | 20점 | 14점 (70%) |
| 4. 문서화 | 10점 | 7점 (70%) |
| 5. 구현 효율성 | 5점 | 3점 (60%) |
| **합계** | **100점** | **74점 (74%)** |
| 보너스 | +10점 | - |

### 등급 기준
- **A+ (90점 이상)**: 탁월한 구현, 프로덕션 배포 가능 수준
- **A (80~89점)**: 우수한 구현, 약간의 개선 필요
- **B (74~79점)**: 양호한 구현, 합격 기준 충족
- **C (60~73점)**: 기본 기능 동작하나 품질 개선 필요
- **F (60점 미만)**: 재구현 권장

---

## 평가 체크리스트 요약

### Phase 1: 자동화된 테스트
```bash
# 1. 단위 테스트 실행
./gradlew test

# 2. 커버리지 리포트 생성
./gradlew jacocoTestReport

# 3. 통합 테스트 (4개 시나리오)
./gradlew integrationTest

# 4. 정적 분석
./gradlew sonarqube
```

### Phase 2: 시나리오 검증
```bash
# Docker Compose 실행
docker-compose up -d

# 각 시나리오 트리거
curl -X POST http://localhost:8080/api/v1/scenarios/scenario-1/trigger
curl -X POST http://localhost:8080/api/v1/scenarios/scenario-2/trigger
curl -X POST http://localhost:8080/api/v1/scenarios/scenario-3/trigger
curl -X POST http://localhost:8080/api/v1/scenarios/scenario-4/trigger

# 장애 감지 확인 (60초 이내)
curl http://localhost:8080/api/v1/incidents

# AI 분석 리포트 확인 (120초 이내)
curl http://localhost:8080/api/v1/incidents/{id}/report

# Slack 알림 확인 (수동)
```

### Phase 3: 성능 측정
```bash
# Kafka 메시지 생성 (부하 테스트)
kafka-console-producer --bootstrap-server localhost:9092 --topic metrics

# JMeter 부하 테스트
jmeter -n -t load-test.jmx -l results.jtl

# Prometheus 메트릭 확인
curl http://localhost:8080/actuator/prometheus
```

### Phase 4: 코드 리뷰
- 아키텍처 일관성 확인
- 에러 핸들링 적절성
- 보안 취약점 스캔
- 코드 스타일 점검

### Phase 5: 문서 검토
- API 문서 완성도 (Swagger UI)
- README 실행 가능 여부
- 트러블슈팅 가이드 유용성

---

## 평가 리포트 템플릿

평가 완료 후 `evaluations/{implementation}/review.md`에 다음 형식으로 작성:

```markdown
# {Implementation Name} - 평가 리포트

**평가 일자**: 2026-02-XX
**평가자**: {Name}
**AI 도구**: {Claude Sonnet 4.5 / GPT-4 / etc.}

## 점수 요약

| 카테고리 | 배점 | 획득 점수 | 비율 |
|----------|------|-----------|------|
| 기능 완성도 | 40점 | XX점 | XX% |
| 코드 품질 | 25점 | XX점 | XX% |
| 비기능 요구사항 | 20점 | XX점 | XX% |
| 문서화 | 10점 | XX점 | XX% |
| 구현 효율성 | 5점 | XX점 | XX% |
| 보너스 | +10점 | XX점 | - |
| **총점** | **100점** | **XX점** | **등급: X** |

## 상세 평가

### 1. 기능 완성도 (XX/40)
- [x] 항목1 (3점)
- [ ] 항목2 (2점) - **미구현**: 이유...

### 2. 코드 품질 (XX/25)
...

## 주요 강점
1. ...
2. ...

## 개선 필요 사항
1. ...
2. ...

## 특이 사항
- ...

## 총평
{전반적인 평가 및 프로덕션 배포 가능 여부}
```

---

이 평가 기준을 통해 모든 AI 구현을 공정하고 객관적으로 비교할 수 있습니다.
