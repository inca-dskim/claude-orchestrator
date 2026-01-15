# Claude Project Orchestrator

프로젝트 요구사항을 분석하여 최적의 Claude Code 설정을 자동으로 생성하는 오케스트레이터입니다.

## 특징

- **맞춤형 설정 생성**: 프로젝트 기술 스택에 맞는 CLAUDE.md, 슬래시 커맨드, 서브에이전트, 스킬, 훅 생성
- **프리뷰 & 선택**: 생성될 항목을 미리 보여주고, 원하는 것만 선택 가능
- **유연한 옵션**: 최소/권장/전체/직접 선택 중 선택
- **예외 처리**: 빈 응답, 불완전한 입력에도 기본값으로 진행

## 설치

```bash
git clone https://github.com/inca-dskim/claude-orchestrator.git
cd claude-orchestrator
```

### Windows 사용자 주의사항

Windows 환경에서 Python 스크립트 실행 시 인코딩 문제가 발생할 수 있습니다. 다음 방법 중 하나를 사용하세요:

```bash
# 방법 1: 환경 변수 설정
set PYTHONIOENCODING=utf-8

# 방법 2: PowerShell에서 실행
$env:PYTHONIOENCODING = "utf-8"

# 방법 3: chcp 명령으로 콘솔 인코딩 변경
chcp 65001
```

## 사용법

### 1. Claude Code 실행

```bash
claude
```

클론한 폴더에서 Claude Code를 실행하면 `CLAUDE.md`가 자동으로 적용됩니다.

### 2. 프로젝트 요구사항 전달

```
Spring Boot 3.4로 REST API 만들어줘
```

### 3. 질문에 답변

오케스트레이터가 다음 항목들을 질문합니다:
- 프로젝트 기본 정보
- 기술 스택
- 아키텍처 (클린 아키텍처/레이어드)
- 데이터베이스
- 인증/보안
- 테스트
- 배포

### 4. 설정 프리뷰에서 선택

```
## 프로젝트 맞춤 Claude 설정 프리뷰

### 슬래시 커맨드 (총 7개)
- ⭐ /build - Gradle 빌드 실행
- ⭐ /test - JUnit 테스트 실행
- ⭐ /commit - 커밋 생성
- /lint - Checkstyle 검사
...

선택해주세요:
1. 최소 - CLAUDE.md만
2. 권장 - CLAUDE.md + ⭐ 표시 항목들
3. 전체 - 위 모든 항목
4. 직접 선택 - 원하는 항목만 골라서
```

### 5. 설정 생성 완료

선택한 항목들이 프로젝트에 생성됩니다.

## 포함된 스킬

| 스킬 | 설명 | 사용법 |
|------|------|--------|
| **skill-creator** | 새로운 스킬 생성 | `/skill-creator` |
| **slash-command-creator** | 슬래시 커맨드 생성 | `/slash-command-creator` |
| **subagent-creator** | 서브에이전트 생성 | `/subagent-creator` |
| **hook-creator** | 훅 생성 | `/hook-creator` |

### 스킬 사용 예시

#### skill-creator 사용 예시
```
/skill-creator

# 대화 예시
사용자: PDF 문서를 다루는 스킬을 만들어줘
Claude: 스킬 이름, 설명, 포함할 기능을 질문...
       → .claude/skills/pdf-handler/ 폴더에 SKILL.md와 스크립트 생성
```

#### slash-command-creator 사용 예시
```
/slash-command-creator

# 대화 예시
사용자: Docker compose up 하는 커맨드 만들어줘
Claude: → .claude/commands/docker-up.md 생성
        이후 /docker-up 으로 사용 가능
```

#### subagent-creator 사용 예시
```
/subagent-creator

# 대화 예시
사용자: 코드 리뷰 전문 에이전트 만들어줘
Claude: 에이전트 역할, 사용할 도구를 질문...
       → .claude/agents/code-reviewer.md 생성
```

#### hook-creator 사용 예시
```
/hook-creator

# 대화 예시
사용자: 파일 저장할 때마다 자동으로 린트 실행하게 해줘
Claude: → .claude/settings.json에 훅 추가
```

### 스킬 유틸리티 스크립트

skill-creator 스킬에는 유용한 Python 스크립트가 포함되어 있습니다:

```bash
# 새 스킬 초기화
python .claude/skills/skill-creator/scripts/init_skill.py my-new-skill --path .claude/skills

# 스킬 유효성 검사
python .claude/skills/skill-creator/scripts/quick_validate.py .claude/skills/my-skill

# 스킬 패키징 (.skill 파일 생성)
python .claude/skills/skill-creator/scripts/package_skill.py .claude/skills/my-skill
```

## 구조

```
claude-orchestrator/
├── CLAUDE.md                 # 메인 오케스트레이터 설정
└── .claude/
    └── skills/
        ├── hook-creator/     # 훅 생성 스킬
        ├── skill-creator/    # 스킬 생성 스킬
        ├── slash-command-creator/  # 커맨드 생성 스킬
        └── subagent-creator/ # 서브에이전트 생성 스킬
```

## 설정 레벨

| 선택 | 생성 항목 | 추천 상황 |
|------|----------|----------|
| **최소** | CLAUDE.md만 | 토이 프로젝트, 토큰 절약 |
| **권장** | CLAUDE.md + 핵심 커맨드/에이전트 | 일반 프로젝트 |
| **전체** | 모든 항목 | 토큰 여유, 최대 퍼포먼스 |
| **직접 선택** | 원하는 것만 | 세밀한 제어 필요 |

## 업데이트

```bash
git pull origin main
```

## 기여

이슈나 PR 환영합니다.

## 라이선스

MIT License
