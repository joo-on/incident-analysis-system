# Functional Requirements

## 1. Overview

Incident Analysis System은 분산 시스템의 실시간 장애 감지 및 AI 기반 근본 원인 분석을 제공하는 플랫폼입니다.

### 1.1 Target Users
- DevOps Engineers
- SRE (Site Reliability Engineers)
- Platform Engineers
- On-call Engineers

### 1.2 Core Value Proposition
- 장애 발생 시 평균 탐지 시간(MTTD) 단축
- AI 기반 근본 원인 분석으로 평균 복구 시간(MTTR) 감소
- 수동 로그 분석 시간 80% 절감

## 2. Functional Requirements

### 2.1 실시간 장애 감지 (Real-time Incident Detection)

#### FR-1: 규칙 기반 모니터링
- **설명**: Kafka로부터 시스템 메트릭 및 로그 스트림을 소비하여 사전 정의된 규칙으로 장애를 감지합니다.
- **입력**: Kafka topic의 메트릭/로그 이벤트
- **출력**: Incident 이벤트 생성
- **처리 지연**: 실시간 (1초 이내)

**감지 규칙 예시**:
| 메트릭 | 임계값 | 조건 | 심각도 |
|--------|--------|------|--------|
| CPU 사용률 | 90% | 5분 이상 지속 | WARNING |
| Memory 사용률 | 95% | 3분 이상 지속 | CRITICAL |
| DB Connection Pool | 사용률 90% | 2분 이상 | CRITICAL |
| Kafka Consumer Lag | 10,000 messages | 즉시 | WARNING |
| HTTP 5xx 에러율 | 5% | 1분 이상 | CRITICAL |
| Response Time | P95 > 3초 | 5분 이상 | WARNING |

#### FR-2: 장애 이벤트 저장
- **설명**: 감지된 장애 정보를 MySQL에 저장하여 이력 관리 및 분석을 지원합니다.
- **저장 항목**:
  - Incident ID (UUID)
  - 발생 시각 (timestamp)
  - 심각도 (CRITICAL, WARNING, INFO)
  - 감지 규칙 ID
  - 영향받은 서비스/호스트
  - 원시 메트릭 데이터
  - 상태 (OPEN, INVESTIGATING, RESOLVED)

#### FR-3: 중복 감지 방지
- **설명**: 동일한 원인의 반복 알림을 그룹화하여 노이즈를 줄입니다.
- **그룹화 조건**:
  - 동일 서비스 + 동일 메트릭
  - 5분 이내 발생
  - 동일 심각도 레벨
- **동작**: 첫 번째 이벤트만 알림, 나머지는 카운트만 증가

### 2.2 AI 기반 근본 원인 분석 (AI-Powered Root Cause Analysis)

#### FR-4: 자동 분석 트리거
- **설명**: CRITICAL 심각도 장애 발생 시 자동으로 AI 분석을 시작합니다.
- **트리거 조건**:
  - 심각도 = CRITICAL
  - 상태 = OPEN
  - 이전 분석 결과 없음
- **실행 타이밍**: 장애 감지 후 10초 이내

#### FR-5: AI 분석 수행
- **설명**: Claude API 또는 OpenAI API를 호출하여 장애 분석을 수행합니다.
- **입력 데이터**:
  - 장애 발생 전후 15분간의 메트릭 데이터
  - 관련 서비스의 로그 (최근 1시간)
  - 시스템 토폴로지 정보
  - 최근 배포 이력 (48시간 이내)
- **AI 프롬프트 구성**:
  ```
  다음 시스템 장애를 분석해주세요:

  [장애 정보]
  - 발생 시각: {timestamp}
  - 영향 서비스: {service_name}
  - 감지된 이상: {anomaly_description}

  [메트릭 데이터]
  {metrics_summary}

  [로그 샘플]
  {log_lines}

  다음 내용을 포함하여 분석해주세요:
  1. 근본 원인 (Root Cause)
  2. 영향 범위 (Impact)
  3. 권장 조치 사항 (Recommended Actions)
  4. 예방 방법 (Prevention)
  ```
- **출력**: 구조화된 분석 리포트 (JSON)

#### FR-6: 분석 리포트 생성
- **설명**: AI 분석 결과를 읽기 쉬운 리포트로 변환합니다.
- **리포트 구조**:
  ```json
  {
    "incident_id": "uuid",
    "analyzed_at": "timestamp",
    "root_cause": {
      "summary": "요약",
      "details": "상세 설명",
      "confidence": 0.95
    },
    "impact": {
      "affected_services": ["service1", "service2"],
      "estimated_users": 1000,
      "severity": "CRITICAL"
    },
    "recommended_actions": [
      {
        "priority": 1,
        "action": "조치 내용",
        "expected_result": "예상 결과"
      }
    ],
    "timeline": [
      {
        "time": "timestamp",
        "event": "이벤트 설명"
      }
    ],
    "prevention": "예방 방법"
  }
  ```
- **저장**: MySQL에 저장 및 Redis 캐싱

#### FR-7: 분석 결과 재시도
- **설명**: AI API 호출 실패 시 재시도 로직을 적용합니다.
- **재시도 정책**:
  - 최대 3회 재시도
  - Exponential backoff (1초, 2초, 4초)
  - 실패 시 Manual Investigation 상태로 전환

### 2.3 수동 조사 모드 (On-demand Investigation)

#### FR-8: Web UI 로그 검색
- **설명**: 사용자가 특정 기간/서비스의 로그를 검색할 수 있습니다.
- **검색 필터**:
  - 시간 범위 (from ~ to)
  - 서비스 이름
  - 로그 레벨 (ERROR, WARN, INFO, DEBUG)
  - 키워드 (정규식 지원)
  - 호스트명
- **결과**:
  - 페이지네이션 (50개/페이지)
  - 로그 라인 하이라이팅
  - 다운로드 기능 (CSV, JSON)

#### FR-9: 서비스 상태 조회
- **설명**: 특정 서비스의 현재 및 과거 메트릭을 조회합니다.
- **조회 가능 메트릭**:
  - CPU, Memory, Disk 사용률
  - Network I/O
  - Database Connection Pool
  - Kafka Consumer Lag
  - HTTP Request Rate, Error Rate
  - Response Time (P50, P95, P99)
- **시각화**: Time-series 그래프 (Chart.js 또는 D3.js)

#### FR-10: 수동 AI 분석 요청
- **설명**: 사용자가 수동으로 AI 분석을 트리거할 수 있습니다.
- **입력**:
  - Incident ID 또는
  - 시간 범위 + 서비스 선택
- **동작**: FR-5와 동일한 AI 분석 수행
- **결과**: 리포트 생성 및 표시

### 2.4 Slack 알림 (Slack Integration)

#### FR-11: CRITICAL 장애 즉시 알림
- **설명**: CRITICAL 심각도 장애 발생 시 Slack으로 알림을 전송합니다.
- **트리거**: CRITICAL 장애 감지 시 (중복 제거 적용)
- **메시지 포맷**:
  ```
  🚨 CRITICAL Incident Detected

  Incident ID: INC-2026-0214-001
  Service: payment-service
  Issue: Database Connection Pool Exhausted
  Detected: 2026-02-14 14:32:15 KST

  📊 Metrics:
  - DB Pool Usage: 98%
  - Active Connections: 49/50
  - Pending Requests: 127

  🔗 View Details: https://incident-system.com/incidents/INC-2026-0214-001
  ```
- **전송 방식**: Slack Webhook API

#### FR-12: AI 분석 완료 알림
- **설명**: AI 분석이 완료되면 결과를 Slack으로 전송합니다.
- **메시지 포맷**:
  ```
  ✅ AI Analysis Completed

  Incident ID: INC-2026-0214-001

  🎯 Root Cause:
  Connection leak in UserService.getUserDetails()
  - Missing connection close in exception handler

  💡 Recommended Actions:
  1. Restart payment-service (immediate)
  2. Deploy hotfix PR#1234 (within 1 hour)
  3. Add connection pool monitoring alert

  📊 View Full Report: [Link]
  ```

#### FR-13: 알림 설정 관리
- **설명**: Web UI에서 알림 설정을 관리할 수 있습니다.
- **설정 항목**:
  - Slack Webhook URL
  - 알림 받을 채널
  - 알림 심각도 필터 (CRITICAL만 또는 WARNING 포함)
  - 알림 시간대 (업무시간만 또는 24/7)
  - 음소거 기간 설정

### 2.5 시스템 검증 (Scenario-based Validation)

#### FR-14: 실패 시나리오 시뮬레이션
- **설명**: 4개의 사전 정의된 실패 시나리오를 시뮬레이션할 수 있습니다.
- **시나리오 목록**:
  1. Database Connection Pool Exhaustion
  2. Kafka Consumer Lag Spike
  3. Memory Leak Detection
  4. Cascading Service Failure
- **시뮬레이션 방법**:
  - API 엔드포인트 호출 (`POST /api/v1/scenarios/{scenario-id}/trigger`)
  - 실제 메트릭/로그 패턴 생성
- **검증 기준**:
  - 장애 감지 시간 < 60초
  - AI 분석 완료 시간 < 120초
  - Slack 알림 전송 성공
  - 정확한 근본 원인 식별 (수동 검증)

## 3. Non-Functional Requirements

### 3.1 Performance
- **NFR-1**: 실시간 이벤트 처리 지연 < 1초 (P95)
- **NFR-2**: AI 분석 완료 시간 < 2분 (P95)
- **NFR-3**: API 응답 시간 < 500ms (P95, 조회 API)
- **NFR-4**: Kafka 메시지 처리량 > 10,000 msg/sec

### 3.2 Scalability
- **NFR-5**: Horizontal scaling 지원 (Spring Boot 인스턴스 추가)
- **NFR-6**: Kafka Consumer Group 기반 병렬 처리
- **NFR-7**: Redis 캐싱으로 DB 부하 분산

### 3.3 Reliability
- **NFR-8**: 시스템 가용성 > 99.5%
- **NFR-9**: AI API 장애 시 Graceful degradation (수동 분석 모드)
- **NFR-10**: 데이터 손실 방지 (Kafka retention 7일, DB 백업 일 1회)

### 3.4 Security
- **NFR-11**: API 인증/인가 (JWT 토큰 기반)
- **NFR-12**: Slack Webhook URL 암호화 저장
- **NFR-13**: AI API Key 환경변수 관리 (절대 코드에 하드코딩 금지)
- **NFR-14**: SQL Injection 방지 (Prepared Statement 사용)

### 3.5 Maintainability
- **NFR-15**: 테스트 커버리지 > 70%
- **NFR-16**: 로깅 표준화 (Logback + JSON format)
- **NFR-17**: API 문서화 (Swagger/OpenAPI)
- **NFR-18**: Database 마이그레이션 관리 (Flyway)

### 3.6 Usability
- **NFR-19**: 반응형 Web UI (모바일 지원)
- **NFR-20**: 다국어 지원 (한국어, 영어)
- **NFR-21**: 직관적인 대시보드 (장애 현황 한눈에 파악)

## 4. Data Requirements

### 4.1 Data Retention
- **Incident 이벤트**: 90일 (이후 아카이빙)
- **AI 분석 리포트**: 180일
- **메트릭 데이터**: 30일 (1분 단위 원본), 1년 (1시간 단위 집계)
- **로그 데이터**: Kafka 7일, 장기 보관은 외부 로그 시스템 (scope 외)

### 4.2 Data Volume (예상)
- **일일 Incident 이벤트**: ~500건
- **일일 메트릭 포인트**: ~100만 건
- **일일 로그 라인**: ~1000만 건
- **AI API 호출**: ~50회/일 (CRITICAL 장애 기준)

## 5. Integration Requirements

### 5.1 External Systems
- **Claude API**: 분석 우선 사용, 실패 시 OpenAI로 Fallback
- **Slack API**: Webhook 기반 알림
- **Kafka**: 메트릭/로그 스트림 소비

### 5.2 API Contracts
- **Claude API**:
  - Model: claude-3-5-sonnet-20240620 (or latest)
  - Max tokens: 4096
  - Temperature: 0.3 (일관된 분석 위해)
- **OpenAI API**:
  - Model: gpt-4-turbo-preview
  - Max tokens: 4096
  - Temperature: 0.3

## 6. Out of Scope (v1.0)

다음 기능들은 초기 버전(v1.0)에서 제외됩니다:
- ❌ 다중 테넌시 (Multi-tenancy)
- ❌ 장애 예측 (Predictive Analytics)
- ❌ 자동 복구 (Auto-remediation)
- ❌ MSTeams, PagerDuty 등 다른 알림 채널
- ❌ 커스텀 감지 규칙 생성 UI (코드로만 가능)
- ❌ 실시간 협업 기능 (채팅, 댓글 등)
- ❌ RBAC (Role-Based Access Control) - 모든 사용자 동일 권한

## 7. Acceptance Criteria

모든 AI 구현은 다음 기준을 충족해야 합니다:

### 7.1 기능 완성도
- [ ] 4개 실패 시나리오 모두 정상 감지
- [ ] AI 분석 성공률 > 95%
- [ ] Slack 알림 전송 성공률 > 99%
- [ ] Web UI 모든 화면 구현 완료

### 7.2 코드 품질
- [ ] 모든 API 엔드포인트에 단위 테스트 존재
- [ ] 통합 테스트로 4개 시나리오 자동 검증
- [ ] SonarQube 품질 게이트 통과 (Code Smells < 50)
- [ ] 보안 취약점 0건 (OWASP 기준)

### 7.3 문서화
- [ ] API 문서 (Swagger UI)
- [ ] 실행 방법 README
- [ ] 아키텍처 다이어그램
- [ ] 트러블슈팅 가이드

### 7.4 운영 준비도
- [ ] Docker Compose로 로컬 실행 가능
- [ ] 헬스체크 엔드포인트 구현
- [ ] 구조화된 로깅 (JSON format)
- [ ] 메트릭 수집 (Micrometer + Prometheus)
