# Implementations

각 AI 도구 및 버전별 구현 결과를 저장하는 디렉토리입니다.

## 명명 규칙

```
{ai-tool}-{model}-{YYYYMMDD}/
```

### 예시
- `claude-sonnet-4.5-20260214/` - Claude Sonnet 4.5, 2026년 2월 14일 구현
- `claude-opus-4.6-20260301/` - Claude Opus 4.6, 2026년 3월 1일 구현
- `gpt-4-20260215/` - GPT-4, 2026년 2월 15일 구현
- `cursor-20260220/` - Cursor IDE, 2026년 2월 20일 구현

## 구현 폴더 구조

각 구현 폴더는 완전한 독립 프로젝트여야 합니다:

```
{implementation-name}/
├── backend/
├── frontend/
├── infra/
├── IMPLEMENTATION.md     # 구현 과정 기록
└── README.md             # 실행 방법
```

## 원칙

- ✅ 각 구현은 완전히 독립적으로 실행 가능해야 합니다
- ✅ 동일한 스펙(`/specs`)을 따라야 합니다
- ✅ 구현 과정, 발생한 문제, 해결 방법을 `IMPLEMENTATION.md`에 기록합니다
- ❌ 다른 구현 폴더의 코드를 참조하거나 의존하지 않습니다
