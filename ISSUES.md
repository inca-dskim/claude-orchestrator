# 발견된 버그 및 개선사항

테스트 일자: 2026-01-15

---

## 버그 (Bugs)

### 1. [Bug] Python 스크립트 Windows 인코딩 에러 (cp949)

**심각도**: High

**설명**: Windows 환경에서 Python 스크립트 실행 시 인코딩 에러가 발생합니다.

**영향받는 파일**:
- `.claude/skills/skill-creator/scripts/init_skill.py`
- `.claude/skills/skill-creator/scripts/package_skill.py`
- `.claude/skills/skill-creator/scripts/quick_validate.py`

**에러 메시지**:

init_skill.py / package_skill.py:
```
UnicodeEncodeError: 'cp949' codec can't encode character '\U0001f680' in position 0: illegal multibyte sequence
```

quick_validate.py:
```
UnicodeDecodeError: 'cp949' codec can't decode byte 0xe2 in position 619: illegal multibyte sequence
```

**원인**:
1. `print()` 문에서 이모지(🚀, 📦, ✅, ❌ 등) 사용 시 Windows cp949 인코딩에서 출력 불가
2. `read_text()` 호출 시 encoding 파라미터 미지정으로 시스템 기본 인코딩(cp949) 사용

**해결 방안**:
```python
# 방안 1: 파일 읽기 시 인코딩 명시
content = skill_md.read_text(encoding='utf-8')

# 방안 2: 이모지를 텍스트로 대체
print("[OK] Created skill directory")  # ✅ 대신
print("[ERROR] Failed")  # ❌ 대신

# 방안 3: stdout 인코딩 설정 (스크립트 상단)
import sys
sys.stdout.reconfigure(encoding='utf-8', errors='replace')
```

---

### 2. [Bug] quick_validate.py가 skill-creator 검증 시 실패

**심각도**: Medium

**설명**: quick_validate.py가 자기 자신(skill-creator)을 검증할 때 파일 인코딩 문제로 실패합니다.

**재현 방법**:
```bash
python .claude/skills/skill-creator/scripts/quick_validate.py .claude/skills/skill-creator
```

**예상 결과**: "Skill is valid!"
**실제 결과**: UnicodeDecodeError

---

## 개선사항 (Enhancements)

### 3. [Enhancement] 프로젝트 생성 경로 지정 기능

**우선순위**: Medium

**설명**: 현재 오케스트레이터는 설정만 생성하고, 실제 프로젝트 코드 생성 시 경로를 지정하는 방법이 명확하지 않습니다.

**제안**:
- Phase 1에 "프로젝트 생성 경로" 질문 추가
- 기본값: 현재 디렉토리 또는 새 하위 디렉토리

---

### 4. [Enhancement] 기술 스택별 템플릿 프리셋

**우선순위**: Low

**설명**: 자주 사용되는 기술 스택 조합에 대한 프리셋을 제공하면 질문 단계를 줄일 수 있습니다.

**제안 프리셋**:
- Spring Boot + JPA + PostgreSQL
- React + TypeScript + Vite
- Node.js + Express + MongoDB
- Python + FastAPI + SQLAlchemy

**사용 예시**:
```
"Spring Boot 프리셋으로 시작할까요? (Yes/No/커스텀)"
```

---

### 5. [Enhancement] 다국어 지원 (i18n)

**우선순위**: Low

**설명**: 현재 CLAUDE.md와 스킬 설명이 한국어로 되어 있어 영어 사용자에게 불편할 수 있습니다.

**제안**:
- CLAUDE.md 영어 버전 제공 (CLAUDE.en.md)
- 또는 언어 선택 옵션 추가

---

### 6. [Enhancement] 생성된 설정 미리보기 (Dry-run)

**우선순위**: Medium

**설명**: 실제 파일 생성 전에 어떤 파일들이 어디에 생성될지 미리 보여주면 좋겠습니다.

**제안**:
```
## 생성 예정 파일 미리보기

다음 파일들이 생성됩니다:
- ./CLAUDE.md
- ./.claude/commands/build.md
- ./.claude/commands/test.md
- ./.claude/agents/code-reviewer.md

진행하시겠습니까? (Yes/No)
```

---

### 7. [Enhancement] 버전 업데이트 알림

**우선순위**: Low

**설명**: 저장소가 업데이트되었을 때 사용자에게 알림을 제공하면 좋겠습니다.

**제안**:
- CLAUDE.md에 버전 체크 로직 추가
- 또는 README에 최신 버전 배지 추가

---

## 문서 개선 (Documentation)

### 8. [Docs] README에 Windows 사용자 주의사항 추가

**설명**: Windows에서 발생할 수 있는 인코딩 문제에 대한 주의사항을 README에 추가해야 합니다.

---

### 9. [Docs] 스킬 사용법 예시 추가

**설명**: 각 스킬(hook-creator, skill-creator 등)의 실제 사용 예시를 README에 추가하면 좋겠습니다.

---

## 테스트 결과 요약

| 항목 | 결과 |
|------|------|
| git clone | ✅ 성공 |
| 파일 구조 | ✅ 정상 |
| init_skill.py | ❌ 인코딩 에러 (Windows) |
| package_skill.py | ❌ 인코딩 에러 (Windows) |
| quick_validate.py | ⚠️ 일부 스킬 검증 실패 |
| init_command.py | ✅ 성공 |
| CLAUDE.md 내용 | ✅ 정상 |
| README.md | ✅ 정상 |
