# Incident Analysis System

A real-time incident detection and AI-powered analysis platform for monitoring distributed systems.

## Overview

This system provides automated incident detection with rule-based monitoring, AI-driven root cause analysis, and comprehensive reporting capabilities. It's designed to help operations teams quickly identify, analyze, and respond to system failures.

### Key Features

- **Real-time Incident Detection**: Rule-based monitoring that generates incident events from system metrics and logs
- **AI-Powered Analysis**: Leverages Claude/OpenAI APIs to analyze incidents and generate detailed reports with root cause analysis
- **On-demand Investigation**: Manual analysis mode with log search and service status inspection through Web UI
- **Slack Integration**: Instant notifications for critical incidents
- **Scenario-based Validation**: 3-4 predefined failure scenarios for system verification and testing

## System Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Log Sources   │────▶│     Kafka       │────▶│  Detection      │
│   & Metrics     │     │   (Streaming)   │     │  Engine         │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
                                                         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Web UI        │◀────│   Spring Boot   │◀────│  Incident       │
│   (React/Vue)   │     │   API Server    │     │  Events         │
└─────────────────┘     └────────┬────────┘     └─────────────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
             ┌──────────┐ ┌──────────┐ ┌──────────┐
             │  MySQL   │ │  Redis   │ │ AI API   │
             │  (Data)  │ │ (Cache)  │ │(Analysis)│
             └──────────┘ └──────────┘ └──────────┘
                                              │
                                              ▼
                                       ┌──────────┐
                                       │  Slack   │
                                       │  Alert   │
                                       └──────────┘
```

## Tech Stack

| Category | Technology |
|----------|------------|
| Backend | Spring Boot, Java |
| Database | MySQL, Flyway (migrations) |
| Messaging | Apache Kafka |
| Cache | Redis |
| AI | Claude API / OpenAI API |
| Alerting | Slack Webhook |
| Frontend | React or Vue.js |
| Infrastructure | Docker Compose |

## Project Structure

```
incident-analysis-system/
├── backend/
│   ├── api/                 # REST API endpoints
│   ├── detection/           # Rule-based detection engine
│   ├── analysis/            # AI analysis integration
│   └── notification/        # Slack notification service
├── frontend/                # Web UI application
├── infra/
│   └── docker-compose.yml   # Local development environment
├── docs/
│   └── scenarios/           # Failure scenario documentation
└── README.md
```

## Failure Scenarios

The system will be validated against these predefined scenarios:

1. **Database Connection Pool Exhaustion** - Simulates connection leak and pool saturation
2. **Kafka Consumer Lag Spike** - Message processing delays causing backpressure
3. **Memory Leak Detection** - Gradual memory consumption leading to OOM
4. **Cascading Service Failure** - Downstream service timeout causing upstream failures

## Roadmap

| Phase | Milestone | Target |
|-------|-----------|--------|
| 1 | Design & Infrastructure Setup | Feb 14, 2026 |
| 2 | Detection Engine Implementation | Feb 28, 2026 |
| 3 | AI Analysis & Report Generation | Mar 14, 2026 |
| 4 | Web UI, Log Search & Alert Configuration | Mar 21, 2026 |
| 5 | Scenario Validation & Documentation | Apr 30, 2026 |

## Getting Started

> Coming soon - Docker Compose setup for local development

```bash
# Clone the repository
git clone https://github.com/your-username/incident-analysis-system.git

# Start infrastructure
docker-compose up -d

# Run the application
./gradlew bootRun
```

## License

MIT License
