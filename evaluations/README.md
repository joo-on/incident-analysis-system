# Evaluations

각 AI 도구 구현에 대한 평가 및 비교 분석 결과를 저장합니다.

## 평가 항목

### 1. 정량적 메트릭
- 코드 라인 수 (LOC)
- 순환 복잡도 (Cyclomatic Complexity)
- 테스트 커버리지
- 빌드 시간
- 구현 완료까지 소요 시간
- AI와의 대화 턴 수

### 2. 정성적 평가
- 코드 품질 (가독성, 유지보수성)
- 아키텍처 일관성
- 베스트 프랙티스 준수
- 에러 핸들링 완성도
- 문서화 수준

### 3. 기능 검증
- 4개 실패 시나리오 통과 여부
- API 스펙 준수도
- 성능 요구사항 충족

## 폴더 구조

```
evaluations/
├── {implementation-name}/
│   ├── metrics.md          # 정량적 메트릭
│   ├── test-results.md     # 시나리오 테스트 결과
│   └── review.md           # 정성적 코드 리뷰
├── comparative-analysis.md # AI 도구 간 비교
└── progress-tracking.md    # 시간에 따른 AI 발전 추적
```

## 평가 프로세스

1. 구현 완료 후 즉시 평가 시작
2. `/specs/evaluation-criteria.md` 기준 적용
3. 모든 평가 결과를 문서화
4. 2개 이상 구현 완료 시 비교 분석 작성
