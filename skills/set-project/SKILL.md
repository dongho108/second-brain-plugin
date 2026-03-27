---
name: set-project
description: |
  현재 작업 중인 프로젝트를 Second Brain vault의 projects/ 폴더에 등록하는 스킬.
  프로젝트 소스에서 맥락을 자동 수집하고, vault 규칙에 맞는 폴더 구조를 생성한다.
  다음 상황에서 트리거한다:
  - 명시적 요청: "프로젝트 등록", "프로젝트 세팅", "set-project", "register project"
  - 프로젝트를 vault에 추가하고 싶을 때: "이 프로젝트 vault에 넣어줘", "프로젝트 연결", "vault에 등록"
---

# Set Project

현재 작업 디렉토리의 프로젝트를 Second Brain vault에 등록한다.
README, package.json, CLAUDE.md, git 정보 등을 자동 수집하여 프로젝트 폴더 구조를 생성한다.

## 실행 순서

### 1. Vault 경로 확인

`$ARGUMENTS`로 vault 경로가 전달되면 사용하고, 없으면 자동 탐지를 시도한다.

자동 탐지 순서:
1. `~/second-brain`에 `projects/` 폴더가 존재하는지 확인
2. 존재하면 `VAULT_PATH="$HOME/second-brain"`

모두 실패하면 AskUserQuestion으로 입력받는다:

```
AskUserQuestion:
  question: "Second Brain vault 경로를 알려주세요."
  header: "Vault 경로"
  options:
    - label: "~/second-brain (기본값)"
      description: "홈 디렉토리 아래 second-brain 폴더를 사용합니다."
    - label: "직접 입력"
      description: "원하는 vault 경로를 직접 지정합니다."
    - label: "/init 으로 세팅하기"
      description: "아직 vault가 없다면 /init 으로 먼저 세팅합니다."
```

- 기본값 선택 시: `VAULT_PATH="$HOME/second-brain"`
- 직접 입력 시: 사용자가 입력한 경로를 `VAULT_PATH`로 사용
- /init 선택 시: "/init 스킬을 먼저 실행해주세요." 안내 후 종료

vault 경로에 `projects/` 폴더가 없으면 에러:

```
❌ $VAULT_PATH/projects/ 가 존재하지 않습니다. /init 으로 vault를 먼저 세팅해주세요.
```

### 2. 프로젝트 정보 자동 수집

현재 작업 디렉토리(`$CWD`)에서 다음 정보를 수집한다:

#### 2-1. 프로젝트명 결정

우선순위:
1. `package.json`의 `name` 필드 (존재할 경우)
2. 현재 디렉토리명 (`basename "$CWD"`)

```bash
if [ -f package.json ]; then
  PROJECT_NAME=$(cat package.json | grep '"name"' | head -1 | sed 's/.*: *"\(.*\)".*/\1/' | sed 's/@[^/]*\///')
fi
PROJECT_NAME="${PROJECT_NAME:-$(basename "$CWD")}"
```

수집한 프로젝트명을 사용자에게 확인한다:

```
AskUserQuestion:
  question: "프로젝트명을 '{PROJECT_NAME}'으로 등록할까요?"
  header: "프로젝트명"
  options:
    - label: "{PROJECT_NAME} (자동 감지)"
      description: "자동으로 감지된 프로젝트명을 사용합니다."
    - label: "직접 입력"
      description: "다른 프로젝트명을 직접 지정합니다."
```

이미 `$VAULT_PATH/projects/$PROJECT_NAME/`이 존재하면:

```
⚠️ 이미 '{PROJECT_NAME}' 프로젝트가 vault에 존재합니다.
기존 프로젝트를 업데이트하려면 /load {PROJECT_NAME} 으로 로드하세요.
```

안내 후 종료.

#### 2-2. 프로젝트 설명 수집

다음 소스에서 순서대로 탐색하여 프로젝트 개요 정보를 수집한다:

```bash
# README.md — 프로젝트 설명의 주요 소스
README_CONTENT=""
if [ -f README.md ]; then
  README_CONTENT=$(cat README.md)
fi

# package.json — name, description, version
PKG_INFO=""
if [ -f package.json ]; then
  PKG_INFO=$(cat package.json)
  # name, description, version, scripts 키 추출
fi

# CLAUDE.md — 프로젝트 컨텍스트
CLAUDE_CONTENT=""
if [ -f CLAUDE.md ]; then
  CLAUDE_CONTENT=$(cat CLAUDE.md)
fi

# .git — remote URL, 최근 커밋 5개
GIT_INFO=""
if [ -d .git ]; then
  GIT_REMOTE=$(git remote get-url origin 2>/dev/null)
  GIT_RECENT=$(git log --oneline -5 2>/dev/null)
fi
```

모든 소스를 읽되, agents.md 작성에 필요한 핵심 정보만 추출한다.

### 3. 프로젝트 폴더 구조 생성

vault의 규칙에 따라 다음 구조를 생성한다:

```
projects/{PROJECT_NAME}/
├── agents.md           # 프로젝트 개요 (5단계에서 작성)
├── claude.md           # → agents.md 심볼릭 링크
├── todo/
│   └── tasks.md        # frontmatter + 섹션 구조
├── lesson-learns/
│   ├── agents.md       # lesson-learns 안내 문서
│   └── claude.md       # → agents.md 심볼릭 링크
└── knowledges/
    ├── agents.md       # knowledges 안내 문서
    └── claude.md       # → agents.md 심볼릭 링크
```

```bash
PROJECT_DIR="$VAULT_PATH/projects/$PROJECT_NAME"

# 디렉토리 생성
mkdir -p "$PROJECT_DIR/todo"
mkdir -p "$PROJECT_DIR/lesson-learns"
mkdir -p "$PROJECT_DIR/knowledges"

# 심볼릭 링크 생성 (agents.md → claude.md)
# 프로젝트 루트
cd "$PROJECT_DIR"
ln -s agents.md claude.md

# lesson-learns
cd "$PROJECT_DIR/lesson-learns"
ln -s agents.md claude.md

# knowledges
cd "$PROJECT_DIR/knowledges"
ln -s agents.md claude.md
```

**핵심 규칙**: `agents.md`를 먼저 생성한 뒤 `claude.md`를 심볼릭 링크로 만든다. 순서가 바뀌면 안 된다.

### 4. Todo 정보 수집 및 tasks.md 생성

#### 4-1. 기존 todo 파일 탐색

프로젝트 소스에서 todo 관련 파일을 찾는다:

```bash
TODO_FILES=$(find "$CWD" -maxdepth 2 \( -iname "todo.md" -o -iname "tasks.md" -o -iname "TODO" \) -not -path "*/node_modules/*" -not -path "*/.git/*" 2>/dev/null)
```

#### 4-2. tasks.md 생성

`$PROJECT_DIR/todo/tasks.md` 파일을 생성한다.

기존 todo 파일이 있으면 내용을 파싱하여 `## Backlog` 섹션에 반영한다.
없으면 빈 템플릿을 생성한다:

```markdown
---
project: {PROJECT_NAME}
updated: 2026-03-27
---

## Backlog

## In Progress

## Done
```

기존 todo에서 가져온 항목이 있으면:

```markdown
---
project: {PROJECT_NAME}
updated: 2026-03-27
---

## Backlog

- [ ] 기존 todo에서 가져온 항목 1
- [ ] 기존 todo에서 가져온 항목 2

## In Progress

## Done
```

frontmatter 규칙:
- `project`: 프로젝트명 (필수)
- `updated`: 마지막 업데이트 날짜 (필수, YYYY-MM-DD 형식)

### 5. agents.md 작성

수집된 정보를 바탕으로 각 agents.md를 작성한다.

#### 5-1. 프로젝트 루트 agents.md

`$PROJECT_DIR/agents.md`에 프로젝트 개요를 작성한다.
파일 구조는 나열하지 않고, 프로젝트의 목적과 맥락을 개략적으로 정리한다:

```markdown
# {PROJECT_NAME}

## 개요

{README.md와 package.json에서 추출한 프로젝트 설명}

## 기술 스택

{package.json의 dependencies, README에서 추출한 기술 정보}

## 저장소

- URL: {git remote URL}
- 소스 경로: {$CWD}

## 최근 작업

{최근 커밋 5개 요약}

## 참고

- CLAUDE.md에서 추출한 핵심 컨텍스트가 있으면 여기에 요약
```

정보가 없는 섹션은 생략한다. 존재하는 정보만으로 작성한다.

#### 5-2. lesson-learns/agents.md

```markdown
# Lesson Learns

{PROJECT_NAME} 프로젝트에서 얻은 교훈과 인사이트를 기록하는 폴더.

## 작성 규칙

- 파일명: `YYYY-MM-DD-주제.md`
- frontmatter: date, context, tags
- 본문: 배운 점을 3-5줄로 간결하게 정리
```

#### 5-3. knowledges/agents.md

```markdown
# Knowledges

{PROJECT_NAME} 프로젝트 관련 지식과 레퍼런스를 모아두는 폴더.

## 작성 규칙

- 프로젝트에 특화된 기술 문서, API 레퍼런스, 설계 결정 등을 저장
- 범용 지식은 common-context/에 저장
```

### 6. 완료 요약 출력

모든 작업이 끝나면 간결하게 요약한다:

```
✅ 프로젝트 등록 완료!
📂 프로젝트: {PROJECT_NAME}
📁 경로: {VAULT_PATH}/projects/{PROJECT_NAME}
📋 todo/tasks.md: {N개 항목 가져옴 / 빈 템플릿 생성}
🔗 심볼릭 링크: 3개 생성 (claude.md → agents.md)

다음 단계:
- /load {PROJECT_NAME} 으로 프로젝트 컨텍스트 로드
- /wrap-up 으로 세션 마무리 시 lesson-learns 자동 저장
```

### 7. 검증

완료 전에 다음을 자동 검증한다:

```bash
PROJECT_DIR="$VAULT_PATH/projects/$PROJECT_NAME"

# 1. 폴더 구조 확인
for dir in "" "/todo" "/lesson-learns" "/knowledges"; do
  [ -d "$PROJECT_DIR$dir" ] || echo "❌ 누락: $PROJECT_DIR$dir"
done

# 2. 심볼릭 링크 확인
for link in "$PROJECT_DIR/claude.md" "$PROJECT_DIR/lesson-learns/claude.md" "$PROJECT_DIR/knowledges/claude.md"; do
  if [ -L "$link" ] && [ "$(readlink "$link")" = "agents.md" ]; then
    echo "✅ $link → agents.md"
  else
    echo "❌ 심볼릭 링크 오류: $link"
  fi
done

# 3. tasks.md frontmatter 확인
head -5 "$PROJECT_DIR/todo/tasks.md" | grep -q "project:" || echo "❌ tasks.md frontmatter 누락: project"
head -5 "$PROJECT_DIR/todo/tasks.md" | grep -q "updated:" || echo "❌ tasks.md frontmatter 누락: updated"

# 4. 필수 파일 존재 확인
for file in "agents.md" "todo/tasks.md" "lesson-learns/agents.md" "knowledges/agents.md"; do
  [ -f "$PROJECT_DIR/$file" ] || echo "❌ 누락: $file"
done
```

검증에서 오류가 발견되면 자동으로 수정을 시도한다.

## $ARGUMENTS 처리

| 입력 | 동작 |
|---|---|
| `/set-project` | vault 자동탐지 → 프로젝트명 자동감지 → 확인 후 등록 |
| `/set-project ~/vault` | 지정 vault → 프로젝트명 자동감지 → 확인 후 등록 |

## 엣지 케이스

- **vault 미발견**: `/init` 스킬 안내
- **이미 등록된 프로젝트**: `/load` 안내 후 종료
- **README/package.json 없음**: 디렉토리명으로 프로젝트명, 최소한의 agents.md 생성
- **git repo가 아닌 경우**: git 관련 정보 없이 나머지만 수집
- **todo 파일 없음**: 빈 템플릿으로 tasks.md 생성
- **vault에 projects/ 폴더 없음**: 에러 메시지 + /init 안내
