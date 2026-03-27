---
name: load
description: |
  Second Brain vault에서 특정 프로젝트의 컨텍스트를 미리 읽어두는 스킬.
  프로젝트의 agents.md, lesson-learns와 공통 컨텍스트를 로드하여 에이전트가 프로젝트 맥락을 이해한 상태에서 작업할 수 있게 한다.
  다음 상황에서 트리거한다:
  - 명시적 요청: "로드", "불러와", "load", "컨텍스트", "context", "작업 시작"
  - 프로젝트 작업 진입: "프로젝트 열어줘", "작업 시작할게", "맥락 파악해줘"
  - 세션 초기화: "이 프로젝트 알아?", "프로젝트 정보 줘"
---

# Load

프로젝트 컨텍스트와 공통 컨텍스트를 미리 읽어 에이전트에게 작업 맥락을 제공한다.

## 실행 순서

### 1. Vault 경로 결정

`$ARGUMENTS`에서 `/` 또는 `~`로 시작하는 인자가 있으면 해당 경로를 `VAULT_PATH`로 사용하고 아래 질문을 건너뛴다.

없으면 현재 디렉토리에 `projects/`와 `common-context/` 폴더가 모두 존재하는지 확인한다:

```bash
if [ -d "projects" ] && [ -d "common-context" ]; then
  VAULT_PATH="$(pwd)"
fi
```

없으면 `~/second-brain`를 시도한다:

```bash
if [ -d "$HOME/second-brain/projects" ] && [ -d "$HOME/second-brain/common-context" ]; then
  VAULT_PATH="$HOME/second-brain"
fi
```

모두 실패하면 AskUserQuestion으로 입력받는다:

```
AskUserQuestion:
  question: "Vault 경로를 찾을 수 없습니다. Second Brain vault 경로를 알려주세요."
  header: "Vault 경로"
  options:
    - label: "직접 입력"
      description: "vault 경로를 직접 지정합니다."
    - label: "/init 으로 세팅하기"
      description: "아직 vault가 없다면 /init 으로 먼저 세팅합니다."
```

- 직접 입력 시: 사용자가 입력한 경로를 `VAULT_PATH`로 사용
- /init 선택 시: "/init 스킬을 먼저 실행해주세요." 안내 후 종료

### 2. 프로젝트 선택

`$ARGUMENTS`에서 경로(`/`, `~` 시작)가 아닌 인자를 프로젝트명으로 추출한다.

프로젝트명이 있으면 `$VAULT_PATH/projects/{프로젝트명}` 폴더가 존재하는지 확인한다. 없으면 유사한 이름의 폴더를 찾아 제안한다.

프로젝트명이 없으면 `$VAULT_PATH/projects/` 하위 폴더를 나열하고 AskUserQuestion으로 선택받는다:

```bash
# 프로젝트 목록 수집
PROJECT_DIRS=$(ls -d "$VAULT_PATH/projects"/*/ 2>/dev/null)
```

```
AskUserQuestion:
  question: "어떤 프로젝트의 컨텍스트를 로드할까요?"
  header: "프로젝트"
  options:
    # 각 프로젝트 폴더에 대해:
    - label: "{프로젝트 폴더명}"
      description: "{agents.md 첫 번째 줄 또는 '설명 없음'}"
    # 최대 4개까지 (AskUserQuestion 제한)
```

프로젝트가 하나도 없으면: "프로젝트가 없습니다. /idea 로 새 프로젝트를 시작해보세요." 안내 후 종료.

선택된 프로젝트명을 `PROJECT_NAME`으로 저장한다.

### 3. 프로젝트 컨텍스트 읽기

#### 3-1. agents.md

```bash
AGENTS_FILE="$VAULT_PATH/projects/$PROJECT_NAME/agents.md"
```

파일이 존재하면 전문을 읽는다. Read 도구를 사용하여 `$AGENTS_FILE`을 읽는다.

파일이 없으면: "⚠️ agents.md가 없습니다. 프로젝트 규칙/목표를 정리해두면 에이전트가 더 정확하게 작업합니다." 경고 후 다음 단계로 진행한다.

#### 3-2. lesson-learns

```bash
LESSON_LEARNS_DIR="$VAULT_PATH/projects/$PROJECT_NAME/lesson-learns"
```

디렉토리가 존재하면 파일을 수정일 역순으로 정렬하여 최근 10개의 `.md` 파일 전문을 읽는다:

```bash
ls -t "$LESSON_LEARNS_DIR"/*.md 2>/dev/null | head -10
```

각 파일을 Read 도구로 읽는다. 10개 초과 시 "📝 최근 10개만 로드했습니다. 전체 {총 개수}개." 안내.

디렉토리가 없으면 조용히 건너뛴다.

#### 3-3. 기타 프로젝트 파일

프로젝트 폴더 내 agents.md와 lesson-learns/ 외의 `.md` 파일 목록만 표시한다:

```bash
find "$VAULT_PATH/projects/$PROJECT_NAME" -maxdepth 1 -name "*.md" ! -name "agents.md" 2>/dev/null
```

파일이 있으면 목록을 출력한다:
```
📄 기타 프로젝트 파일:
  - todo.md
  - ideas.md
  ...
```

### 4. 공통 컨텍스트 읽기

#### 4-1. 공통 lesson-learns

```bash
COMMON_LESSONS_DIR="$VAULT_PATH/common-context/lesson-learns"
```

디렉토리가 존재하면 수정일 역순으로 최근 5개의 `.md` 파일 전문을 읽는다:

```bash
ls -t "$COMMON_LESSONS_DIR"/*.md 2>/dev/null | head -5
```

각 파일을 Read 도구로 읽는다.

디렉토리가 없으면 조용히 건너뛴다.

#### 4-2. 기타 공통 컨텍스트 파일

`common-context/` 내 lesson-learns/ 외의 `.md` 파일 목록만 표시한다:

```bash
find "$VAULT_PATH/common-context" -maxdepth 1 -name "*.md" 2>/dev/null
```

파일이 있으면 목록을 출력한다:
```
📄 공통 컨텍스트 파일:
  - coding-conventions.md
  - review-checklist.md
  ...
```

### 5. 경로 맵 출력

이후 작업(todo 수정, lesson-learns 추가 등)에서 정확한 경로를 사용할 수 있도록 경로 맵을 출력한다:

```
📂 작업 경로 맵
🏠 VAULT_PATH=$VAULT_PATH
📁 PROJECT_PATH=$VAULT_PATH/projects/$PROJECT_NAME
📁 LESSON_LEARNS_DIR=$VAULT_PATH/projects/$PROJECT_NAME/lesson-learns
📁 COMMON_CONTEXT_PATH=$VAULT_PATH/common-context
📁 COMMON_LESSONS_DIR=$VAULT_PATH/common-context/lesson-learns
📁 IDEAS_PATH=$VAULT_PATH/ideas
```

### 6. 컨텍스트 요약

로드된 항목 수를 간결하게 요약한다:

```
✅ 프로젝트 컨텍스트 로드 완료!
📋 프로젝트: $PROJECT_NAME
📖 agents.md: [로드됨/없음]
📝 프로젝트 lesson-learns: N개 로드
📝 공통 lesson-learns: N개 로드
📄 기타 파일: N개 발견
```

### 7. 세션 메모리 저장

이후 대화에서 경로 정보를 유지하기 위해 세션 메모리에 저장한다:

```
<remember priority>
현재 작업 중인 Second Brain 프로젝트 경로 정보:
- VAULT_PATH=$VAULT_PATH
- PROJECT_NAME=$PROJECT_NAME
- PROJECT_PATH=$VAULT_PATH/projects/$PROJECT_NAME
- LESSON_LEARNS_DIR=$VAULT_PATH/projects/$PROJECT_NAME/lesson-learns
- COMMON_CONTEXT_PATH=$VAULT_PATH/common-context
- COMMON_LESSONS_DIR=$VAULT_PATH/common-context/lesson-learns
- IDEAS_PATH=$VAULT_PATH/ideas
</remember>
```

## $ARGUMENTS 처리

| 입력 | 동작 |
|---|---|
| `/load` | vault 자동탐지 → 프로젝트 목록 → 선택 |
| `/load 프로젝트명` | vault 자동탐지 → 해당 프로젝트 바로 로드 |
| `/load ~/vault 프로젝트명` | 지정 vault → 해당 프로젝트 바로 로드 |

## 엣지 케이스

- **vault 미발견**: `/init` 스킬 안내
- **프로젝트 폴더 없음**: `/idea` 스킬 안내
- **agents.md 없음**: 경고 출력 후 나머지 컨텍스트는 정상 로드
- **lesson-learns 과다**: 프로젝트 최근 10개, 공통 최근 5개만 로드
- **빈 프로젝트 폴더**: "컨텍스트 파일이 없습니다." 안내 후 경로 맵만 출력
