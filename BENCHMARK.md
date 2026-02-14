# AI Tools Benchmark Project

이 프로젝트는 **Incident Analysis System** 구현을 통해 다양한 AI 도구의 발전을 측정하고 비교하는 벤치마크 프로젝트입니다.

## 프로젝트 목표

1. **AI 도구 비교**: Claude, GPT, Cursor 등 여러 AI 도구의 코드 생성 능력 비교
2. **시간별 발전 추적**: 동일한 AI 도구의 버전별, 시간별 성능 개선 측정
3. **베스트 프랙티스 발견**: 효과적인 AI 협업 패턴 및 프롬프트 전략 학습

## 벤치마크 방법론

### 1. 고정 변수
- **스펙**: `/specs`의 요구사항, 아키텍처, API 명세는 모든 구현에서 동일
- **테스트 데이터**: `/reference/test-data`의 데이터로 동일 조건 검증
- **평가 기준**: `/specs/evaluation-criteria.md`의 체크리스트 일관 적용

### 2. 독립 변수
- AI 도구 종류 (Claude, GPT, Cursor, etc.)
- AI 모델 버전
- 구현 시점 (시간에 따른 AI 발전)

### 3. 측정 항목
- **정량적**: LOC, 복잡도, 커버리지, 소요 시간, 대화 턴 수
- **정성적**: 코드 품질, 아키텍처, 문서화, 에러 핸들링
- **기능적**: 4개 시나리오 통과율, API 스펙 준수도

## 워크플로우

### Phase 1: 스펙 작성 (완료)
- [x] README.md - 프로젝트 개요
- [ ] specs/requirements.md - 상세 요구사항
- [ ] specs/architecture.md - 아키텍처 설계
- [ ] specs/api-spec.md - API 명세
- [ ] specs/scenarios/ - 4개 실패 시나리오
- [ ] specs/evaluation-criteria.md - 평가 기준

### Phase 2: 첫 구현 (Claude Sonnet 4.5)
```bash
# 1. 구현 폴더 생성
mkdir implementations/claude-sonnet-4.5-20260214

# 2. 프롬프트 기록 시작
echo "# Claude Sonnet 4.5 - 2026-02-14" > prompts/claude-sonnet-4.5-20260214.md

# 3. AI와 협업하여 구현
# - specs/ 문서를 AI에게 전달
# - 대화 과정 prompts/ 파일에 기록

# 4. 구현 완료 후 평가
mkdir evaluations/claude-sonnet-4.5-20260214
# - metrics.md, test-results.md, review.md 작성
```

### Phase 3: 추가 AI 도구로 재구현
- GPT-4, Cursor, Copilot 등으로 동일 스펙 구현
- 각 구현마다 독립적인 폴더 생성
- 동일한 평가 기준 적용

### Phase 4: 비교 분석
- `evaluations/comparative-analysis.md` 작성
- AI 도구별 강점/약점 분석
- 효과적인 프롬프트 패턴 정리

## 폴더 구조

```
incident-analysis-system/
├── specs/                    # 고정 스펙 (AI 독립적)
├── implementations/          # AI별 구현 결과
├── evaluations/             # 평가 및 비교 분석
├── reference/               # 공통 참고자료
├── prompts/                 # 프롬프트 기록
├── README.md                # 프로젝트 소개
├── BENCHMARK.md            # 이 문서
└── CLAUDE.md               # Claude 작업 가이드
```

## 예상 일정

| 날짜 | 작업 | AI 도구 |
|------|------|---------|
| 2026-02-14 | 스펙 작성 완료 | - |
| 2026-02-15 | 첫 구현 시작 | Claude Sonnet 4.5 |
| 2026-02-28 | 첫 구현 완료 및 평가 | Claude Sonnet 4.5 |
| 2026-03-01 | 두 번째 구현 | Claude Opus 4.6 |
| 2026-03-15 | 세 번째 구현 | GPT-4 |
| 2026-03-30 | 비교 분석 완료 | - |

## 성공 기준

- ✅ 최소 3개 이상의 AI 도구로 구현 완료
- ✅ 모든 구현이 4개 시나리오 통과
- ✅ 정량적 메트릭 수집 완료
- ✅ 비교 분석 리포트 작성
- ✅ 효과적인 프롬프트 패턴 문서화

## 학습 목표

1. 각 AI 도구의 강점 파악 (설계, 구현, 디버깅 등)
2. AI 협업 시 효과적인 커뮤니케이션 방법 학습
3. 시간에 따른 AI 기술 발전 속도 체감
4. 실전 프로젝트에서 AI 도구 선택 가이드라인 확립
