# Scenario 3: Memory Leak Detection

## 개요

**시나리오 ID**: `scenario-3`
**시나리오 명**: Memory Leak Detection
**심각도**: WARNING → CRITICAL
**예상 발생 빈도**: 낮음-중간 (월 1회)

### 설명
애플리케이션의 메모리 사용량이 시간이 지남에 따라 점진적으로 증가하여 결국 OutOfMemoryError(OOM)를 유발하는 상황입니다. 일반적으로 객체가 적절히 해제되지 않거나 캐시가 무한정 증가할 때 발생합니다.

### 비즈니스 영향
- 애플리케이션 성능 저하 (GC 빈번 발생)
- 서비스 중단 (OOM으로 인한 크래시)
- 응답 시간 증가
- 고객 경험 악화

---

## 근본 원인 (Root Cause)

### 주요 원인

1. **Unbounded Cache Growth**
   ```java
   // ❌ 크기 제한 없는 캐시
   @Service
   public class IncidentCache {
       // 영원히 증가하는 Map
       private Map<UUID, Incident> cache = new HashMap<>();

       public void put(UUID id, Incident incident) {
           cache.put(id, incident);
           // 제거 로직 없음!
       }
   }
   ```

2. **Collection Memory Leak**
   ```java
   // ❌ Static 컬렉션에 계속 추가
   public class MetricCollector {
       private static List<MetricEvent> allMetrics = new ArrayList<>();

       public void collect(MetricEvent event) {
           allMetrics.add(event);
           // 영원히 메모리에 유지
       }
   }
   ```

3. **Improper Listener/Callback Registration**
   ```java
   // ❌ Listener 등록 후 제거하지 않음
   public void subscribe() {
       eventBus.register(this);
       // unregister() 호출 안 함
   }
   ```

4. **Thread-local Variables Not Cleaned**
   ```java
   // ❌ ThreadLocal 정리 안 함
   private static ThreadLocal<List<String>> context = new ThreadLocal<>();

   public void process() {
       context.set(new ArrayList<>());
       // 사용 후 context.remove() 호출 안 함
   }
   ```

5. **Large Object Retention**
   - 큰 객체를 클래스 필드로 보유
   - 파일 핸들, 네트워크 연결 미해제

---

## 시뮬레이션 방법

### API 트리거

```bash
curl -X POST http://localhost:8080/api/v1/scenarios/scenario-3/trigger \
  -H "Content-Type: application/json" \
  -d '{
    "duration": "PT10M",
    "intensity": "HIGH"
  }'
```

### 시뮬레이션 로직

```java
@Service
public class MemoryLeakScenario {

    // 의도적인 메모리 누수용 컬렉션
    private static List<byte[]> leakedMemory = new ArrayList<>();
    private static Map<String, AnalysisReport> unboundedCache = new HashMap<>();

    public void execute(String intensity) {
        int leakRateMB = intensity.equals("HIGH") ? 10 : 5;
        int intervalSeconds = 10;

        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        scheduler.scheduleAtFixedRate(() -> {
            try {
                // 1. 큰 바이트 배열 생성 및 보유 (GC 불가)
                byte[] leak = new byte[leakRateMB * 1024 * 1024]; // MB 단위
                Arrays.fill(leak, (byte) 1);
                leakedMemory.add(leak);

                // 2. 무제한 캐시에 데이터 추가
                for (int i = 0; i < 1000; i++) {
                    String key = "report-" + UUID.randomUUID();
                    AnalysisReport report = createDummyReport();
                    unboundedCache.put(key, report);
                }

                // 3. ThreadLocal 정리 안 함
                ThreadLocal<List<String>> threadLocal = new ThreadLocal<>();
                threadLocal.set(new ArrayList<>(10000));
                // threadLocal.remove() 호출 안 함

                log.info("Memory leaked: {} MB, Cache size: {}",
                    leakedMemory.size() * leakRateMB,
                    unboundedCache.size());

            } catch (OutOfMemoryError e) {
                log.error("OutOfMemoryError occurred (expected)", e);
                scheduler.shutdown();
            }
        }, 0, intervalSeconds, TimeUnit.SECONDS);

        // 10분 후 정리 (시나리오 종료)
        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                scheduler.shutdown();
                leakedMemory.clear();
                unboundedCache.clear();
                System.gc(); // 명시적 GC 요청
                log.info("Memory leak scenario ended, memory cleared");
            }
        }, 600000); // 10분
    }
}
```

### 생성되는 메트릭 데이터

```json
{
  "timestamp": "2026-02-14T14:30:12.345Z",
  "service": "incident-analysis-api",
  "host": "api-prod-01",
  "metric": "jvm.memory.used",
  "value": 1932735283,
  "unit": "bytes",
  "metadata": {
    "area": "heap",
    "id": "G1 Old Gen",
    "max": 2147483648,
    "committed": 2147483648,
    "used": 1932735283,
    "usage": 89.9
  }
}
```

### 생성되는 로그 데이터

```json
{
  "timestamp": "2026-02-14T14:35:12.345Z",
  "level": "WARN",
  "service": "incident-analysis-api",
  "host": "api-prod-01",
  "logger": "sun.misc.GC",
  "message": "GC overhead limit exceeded",
  "context": {
    "gcType": "G1 Old Generation",
    "gcTime": 8234,
    "gcCount": 156,
    "heapUsage": 95.2
  }
}

{
  "timestamp": "2026-02-14T14:36:45.678Z",
  "level": "ERROR",
  "service": "incident-analysis-api",
  "host": "api-prod-01",
  "logger": "com.incident.analysis.service.AIAnalyzerService",
  "message": "java.lang.OutOfMemoryError: Java heap space",
  "exception": "java.lang.OutOfMemoryError: Java heap space\n\tat java.util.Arrays.copyOf(Arrays.java:3332)\n\tat java.util.ArrayList.grow(ArrayList.java:275)",
  "context": {
    "heapUsage": 98.7,
    "threadCount": 156
  }
}
```

---

## 감지 기준 (Detection Rule)

### 규칙 정의

```java
@Component
public class MemoryLeakRule implements DetectionRule {

    @Override
    public boolean evaluate(MetricEvent event) {
        if (!event.getMetricName().equals("jvm.memory.used")) {
            return false;
        }

        double usagePercent = event.getMetadata().getDouble("usage");

        // 1. 메모리 사용률 95% 초과 (3분 이상)
        if (usagePercent > 95.0 && event.getDuration() > Duration.ofMinutes(3)) {
            return true;
        }

        // 2. 메모리 사용률 지속 증가 패턴 감지
        List<Double> recentUsage = getRecentMemoryUsage(event.getServiceName(), 10);
        return isGradualIncrease(recentUsage);
    }

    private boolean isGradualIncrease(List<Double> values) {
        // 최근 10개 데이터 포인트가 모두 증가 추세
        for (int i = 1; i < values.size(); i++) {
            if (values.get(i) <= values.get(i - 1)) {
                return false;
            }
        }
        return values.size() >= 10 && values.get(values.size() - 1) > 80.0;
    }

    @Override
    public Severity getSeverity() {
        return Severity.CRITICAL;
    }

    @Override
    public String getDescription() {
        return "Memory usage > 95% for 3 minutes or gradual memory leak pattern detected";
    }
}
```

### 임계값

| 메트릭 | 임계값 | 지속 시간 | 심각도 |
|--------|--------|-----------|--------|
| jvm.memory.used | > 95% | 3분 | CRITICAL |
| jvm.memory.used | > 90% | 5분 | WARNING |
| jvm.memory.used | 지속 증가 패턴 (10분) | - | WARNING |

### 예상 감지 시간

- **목표**: 60초 이내 (임계값 도달 후)
- **프로세스**:
  1. JVM 메트릭 수집 (30초 주기)
  2. 패턴 분석 (증가 추세 확인)
  3. Rule Engine 평가 - 1초
  4. Incident 생성 - 1초

---

## AI 분석 예상 결과

### Root Cause Analysis

```json
{
  "root_cause": {
    "summary": "Unbounded cache growth in IncidentCache causing memory leak",
    "details": "The IncidentCache class uses a HashMap without size limits or eviction policy. Analysis of heap dumps shows ~1.2GB occupied by Incident objects in the cache, with 50,000+ entries never being removed. The cache grows at ~100 entries/minute during normal operation, leading to eventual OutOfMemoryError after ~8 hours of uptime.",
    "confidence": 0.92,
    "evidence": [
      "Heap usage increased linearly from 30% to 98% over 6 hours",
      "GC frequency increased from 2/min to 45/min",
      "GC pause time increased from 50ms to 8s (GC overhead limit exceeded)",
      "Heap dump shows HashMap consuming 1.2GB (60% of total heap)",
      "No cache eviction logs found"
    ],
    "leak_location": "com.incident.analysis.cache.IncidentCache.cache (HashMap)"
  }
}
```

### Impact Assessment

```json
{
  "impact": {
    "affected_services": ["incident-analysis-api"],
    "severity": "CRITICAL",
    "business_impact": "Service will crash with OutOfMemoryError in ~30 minutes if not addressed. All API requests timing out due to excessive GC pauses.",
    "performance_degradation": {
      "avgResponseTime": {
        "before": "120ms",
        "current": "4500ms",
        "increase": "37.5x"
      },
      "gcPauseTime": {
        "before": "50ms",
        "current": "8000ms",
        "increase": "160x"
      }
    },
    "downstream_effects": [
      "Web UI unresponsive",
      "Incident detection delayed",
      "AI analysis timing out",
      "Slack notifications failing"
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
      "action": "Restart incident-analysis-api immediately to clear memory",
      "expected_result": "Immediate service recovery, memory reset to baseline (~30%)",
      "estimated_duration": "PT3M",
      "risk": "LOW - Brief service interruption during restart"
    },
    {
      "priority": 2,
      "action": "Deploy hotfix: Add Caffeine cache with max size 10,000 and 1-hour TTL",
      "code_example": "@Bean\npublic Cache<UUID, Incident> incidentCache() {\n  return Caffeine.newBuilder()\n    .maximumSize(10000)\n    .expireAfterWrite(1, TimeUnit.HOURS)\n    .build();\n}",
      "expected_result": "Prevent unbounded cache growth, memory usage stable at ~50%",
      "estimated_duration": "PT1H",
      "risk": "LOW"
    },
    {
      "priority": 3,
      "action": "Increase JVM heap from 2GB to 4GB as temporary mitigation",
      "expected_result": "More headroom before OOM, buys time for permanent fix",
      "estimated_duration": "PT5M",
      "risk": "MEDIUM - Masks root cause, doesn't solve leak"
    },
    {
      "priority": 4,
      "action": "Enable heap dump on OOM: -XX:+HeapDumpOnOutOfMemoryError",
      "expected_result": "Automatic heap dump for forensic analysis if OOM occurs again",
      "estimated_duration": "PT5M",
      "risk": "LOW - Disk space required (~2-4GB)"
    },
    {
      "priority": 5,
      "action": "Add memory monitoring alert at 80% threshold",
      "expected_result": "Early warning before critical level",
      "estimated_duration": "PT30M",
      "risk": "LOW"
    }
  ]
}
```

### Prevention

```
1. Code Review: Enforce bounded caches (Caffeine, Guava) in all services
2. Static Analysis: Add SonarQube rule to detect unbounded collections
3. Load Testing: Include sustained load tests (8+ hours) to catch memory leaks
4. Monitoring: Set up memory usage trend alerts (gradual increase over 4 hours)
5. Heap Profiling: Regular heap dump analysis in staging environment
6. JVM Tuning: Configure appropriate heap size based on load testing results
```

---

## 검증 기준 (Acceptance Criteria)

### 자동화된 검증

```java
@SpringBootTest
public class Scenario3Test {

    @Test
    public void testMemoryLeakDetection() {
        // 1. 시나리오 트리거
        triggerScenario("scenario-3");

        // 2. 메모리 사용률 증가 확인
        await().atMost(5, MINUTES).until(() -> {
            double memoryUsage = getMemoryUsagePercent();
            return memoryUsage > 90.0;
        });

        // 3. 60초 이내 장애 감지 확인 (임계값 도달 후)
        await().atMost(60, SECONDS).until(() -> {
            List<Incident> incidents = incidentRepository
                .findByRuleId("jvm.memory.leak");
            return !incidents.isEmpty();
        });

        Incident incident = incidents.get(0);
        assertThat(incident.getSeverity()).isEqualTo(Severity.CRITICAL);

        // 4. 120초 이내 AI 분석 완료 확인
        await().atMost(120, SECONDS).until(() -> {
            return analysisReportRepository
                .findByIncidentId(incident.getId())
                .isPresent();
        });

        AnalysisReport report = analysisReportRepository
            .findByIncidentId(incident.getId())
            .get();

        // 5. 근본 원인에 "cache" 또는 "leak" 포함 확인
        assertThat(report.getRootCause().getSummary())
            .containsAnyOf("cache", "leak", "unbounded", "growth");

        // 6. 권장 조치에 "cache limit" 또는 "restart" 포함
        assertThat(report.getRecommendedActions())
            .anyMatch(action ->
                action.getAction().containsAnyOf("restart", "cache", "limit", "eviction")
            );

        // 7. Heap size 증가 또는 캐시 제한 권장 확인
        assertThat(report.getRecommendedActions())
            .hasSizeGreaterThan(2);
    }
}
```

### 수동 검증 체크리스트

- [ ] JVM 메모리 사용률이 95% 초과 확인
- [ ] GC 로그에서 "GC overhead limit exceeded" 또는 유사 경고 확인
- [ ] Slack에 CRITICAL 알림 수신
- [ ] Web UI에서 메모리 메트릭 차트 확인 (점진적 증가 패턴)
- [ ] AI 분석에서 캐시 또는 컬렉션 증가 원인 식별
- [ ] 권장 조치에 캐시 제한 또는 재시작 포함
- [ ] 코드 예시 또는 구체적인 수정 방법 제시 확인

---

## 참고 자료

### JVM Memory Monitoring

```bash
# JVM 메모리 사용량 확인
jstat -gc <pid> 1000  # 1초마다 GC 통계

# Heap dump 생성 (분석용)
jmap -dump:live,format=b,file=heap.bin <pid>

# Heap dump 분석 (Eclipse MAT)
java -jar mat.jar heap.bin

# GC 로그 활성화
-Xlog:gc*:file=gc.log:time,uptime,level,tags
```

### Spring Boot Actuator Metrics

```bash
# 메모리 메트릭 조회
curl http://localhost:8080/actuator/metrics/jvm.memory.used
curl http://localhost:8080/actuator/metrics/jvm.memory.max
curl http://localhost:8080/actuator/metrics/jvm.gc.pause

# 메트릭 출력 예시
{
  "name": "jvm.memory.used",
  "measurements": [
    {"statistic": "VALUE", "value": 1932735283}
  ],
  "availableTags": [
    {"tag": "area", "values": ["heap", "nonheap"]},
    {"tag": "id", "values": ["G1 Old Gen", "G1 Eden Space"]}
  ]
}
```

### Caffeine Cache 설정 (권장)

```java
// ✅ 올바른 캐시 설정
@Configuration
public class CacheConfig {

    @Bean
    public Cache<UUID, Incident> incidentCache() {
        return Caffeine.newBuilder()
            .maximumSize(10000)  // 최대 10,000개
            .expireAfterWrite(1, TimeUnit.HOURS)  // 1시간 후 자동 제거
            .expireAfterAccess(30, TimeUnit.MINUTES)  // 30분 미사용 시 제거
            .recordStats()  // 캐시 통계 수집
            .build();
    }

    @Bean
    public CacheMetricsCollector cacheMetrics(Cache<UUID, Incident> cache) {
        return new CacheMetricsCollector(cache, "incident_cache");
    }
}

// 사용 예시
@Service
public class IncidentService {

    private final Cache<UUID, Incident> cache;

    public Incident getIncident(UUID id) {
        return cache.get(id, key -> {
            // 캐시 미스 시 DB에서 조회
            return incidentRepository.findById(key).orElse(null);
        });
    }
}
```

---

## 실제 사례 참고

### Case Study 1: Twitter 2012 Memory Leak
- **원인**: Ruby GC 문제로 인한 메모리 누수
- **영향**: 서비스 지연 및 간헐적 다운
- **해결**: GC 튜닝 + 코드 리팩토링

### Case Study 2: Netflix Hystrix Thread Pool Leak
- **원인**: ThreadLocal 변수 정리 안 함
- **영향**: 장시간 실행 시 메모리 부족
- **해결**: ThreadLocal.remove() 명시적 호출

---

## 추가 고려사항

### Heap Dump 분석

Eclipse MAT (Memory Analyzer Tool) 사용:

1. **Leak Suspects Report**: 자동으로 메모리 누수 후보 식별
2. **Dominator Tree**: 가장 많은 메모리를 차지하는 객체 트리
3. **Histogram**: 클래스별 인스턴스 개수 및 메모리 사용량

```bash
# MAT 쿼리 예시
# 1. 특정 클래스의 모든 인스턴스 찾기
SELECT * FROM java.util.HashMap

# 2. 크기 순으로 정렬
SELECT toString(s), s.@retainedHeapSize
FROM java.util.HashMap s
ORDER BY s.@retainedHeapSize DESC

# 3. Incident 객체 개수 확인
SELECT COUNT(*) FROM com.incident.analysis.entity.Incident
```

### GC Tuning

```bash
# G1GC 사용 (Java 9+, 권장)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45

# Heap 크기 설정
-Xms2g  # 초기 heap
-Xmx4g  # 최대 heap

# OOM 시 heap dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdumps/

# GC 로그
-Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags
```

### 프로덕션 권장사항

```java
// ✅ 안전한 ThreadLocal 사용
public class RequestContext {
    private static ThreadLocal<Map<String, Object>> context =
        ThreadLocal.withInitial(HashMap::new);

    public static void set(String key, Object value) {
        context.get().put(key, value);
    }

    public static void clear() {
        context.remove();  // 필수!
    }
}

// Filter에서 요청 종료 시 정리
@Component
public class RequestContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        try {
            chain.doFilter(request, response);
        } finally {
            RequestContext.clear();  // 반드시 정리
        }
    }
}
```

---

이 시나리오는 점진적으로 발생하는 메모리 누수를 다룹니다. AI가 메모리 증가 패턴을 인식하고, 근본 원인(예: 무제한 캐시)을 식별하며, 즉각적인 조치(재시작)와 장기적인 해결책(캐시 제한)을 모두 제안하는지 평가하는 벤치마크입니다.
