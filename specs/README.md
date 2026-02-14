# Specifications

이 디렉토리는 모든 AI 도구 구현이 따라야 할 **고정된 설계 문서**를 포함합니다.

## 문서 구성

- `requirements.md` - 기능 요구사항 및 제약사항
- `architecture.md` - 시스템 아키텍처 및 컴포넌트 설계
- `api-spec.md` - REST API 명세
- `scenarios/` - 실패 시나리오 상세 정의
- `evaluation-criteria.md` - 구현 평가 기준 및 체크리스트

## 원칙

- ✅ 이 스펙은 AI 도구 독립적입니다
- ✅ 모든 구현은 이 스펙을 100% 따라야 합니다
- ✅ 스펙 변경 시 버전 관리를 통해 추적합니다
- ❌ 특정 AI 도구나 구현 방식에 종속되지 않습니다

## 다음 단계

1. `requirements.md` 작성 - 기능 요구사항 정의
2. `architecture.md` 작성 - 시스템 설계 문서화
3. `scenarios/` 작성 - 4개 실패 시나리오 상세화
4. `evaluation-criteria.md` 작성 - 평가 기준 수립
