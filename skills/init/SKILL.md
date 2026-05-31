---
name: init
description: |
  Second Brain vault를 새 환경에 세팅하는 스킬.
  GitHub에서 repo를 clone하고, git 인증·Obsidian CLI 확인·심볼릭 링크 검증까지 자동으로 수행한다.
  "/init", "세팅", "설치", "setup", "볼트 세팅" 등의 표현에 트리거한다.
---

# Second Brain Init

새로운 환경에서 Second Brain vault를 자동으로 세팅하는 워크플로우.

## 사전 조건

- `github_token`: GitHub PAT (repo 권한 필요) — AskUserQuestion으로 런타임에 입력받음
- `vault_path`: vault를 clone할 경로 (기본값: `~/second-brain`) — AskUserQuestion으로 런타임에 입력받음

## 실행 순서

### 1. 사용자 입력 수집

`$ARGUMENTS`로 vault 경로가 전달되지 않은 경우, AskUserQuestion을 사용하여 필요한 정보를 수집한다.

#### 1-1. Vault 경로

`$ARGUMENTS`가 비어있으면 AskUserQuestion으로 vault 경로를 입력받는다:

```
AskUserQuestion:
  question: "Vault를 어디에 clone할까요?"
  header: "Vault 경로"
  options:
    - label: "~/second-brain (기본값)"
      description: "홈 디렉토리 아래 second-brain 폴더에 clone합니다."
    - label: "직접 입력"
      description: "원하는 경로를 직접 지정합니다."
```

- 기본값 선택 시: `VAULT_PATH="$HOME/second-brain"`
- 직접 입력 시: 사용자가 입력한 경로를 `VAULT_PATH`로 사용
- `$ARGUMENTS`가 있으면: 해당 값을 `VAULT_PATH`로 사용하고 이 질문을 건너뜀

#### 1-2. GitHub Token

AskUserQuestion으로 GitHub PAT를 입력받는다:

```
AskUserQuestion:
  question: "GitHub Personal Access Token을 입력해주세요. (repo 권한이 필요합니다)"
  header: "GitHub Token"
  options:
    - label: "직접 입력"
      description: "GitHub PAT를 입력합니다. Settings > Developer settings > Personal access tokens에서 생성할 수 있습니다."
    - label: "건너뛰기"
      description: "이미 git credential이 설정되어 있다면 건너뛸 수 있습니다."
```

- 직접 입력 시: 사용자가 입력한 값을 `GITHUB_TOKEN`으로 사용
- 건너뛰기 시: 토큰 없이 진행 (기존 git credential에 의존)

### 2. Repo Clone

수집한 값을 사용하여 repo를 clone한다:

```bash
# VAULT_PATH와 GITHUB_TOKEN은 1단계에서 수집한 값

# 이미 존재하면 pull
if [ -d "$VAULT_PATH/.git" ]; then
  echo "기존 vault 발견 — pull로 최신화합니다."
  cd "$VAULT_PATH"
  git pull
else
  if [ -n "$GITHUB_TOKEN" ]; then
    # clone with token
    git clone "https://${GITHUB_TOKEN}@github.com/dongho108/second-brain.git" "$VAULT_PATH"
  else
    # clone without token (기존 credential에 의존)
    git clone "https://github.com/dongho108/second-brain.git" "$VAULT_PATH"
  fi
  cd "$VAULT_PATH"
fi
```

### 3. Git 인증 설정

GitHub 토큰이 제공된 경우, `.envrc`와 `.git-askpass`를 생성하여 이후 push/pull이 자동으로 동작하도록 한다:

```bash
cd "$VAULT_PATH"

# 토큰이 있을 때만 인증 파일 생성
if [ -n "$GITHUB_TOKEN" ]; then
  # .git-askpass 생성
  cat > .git-askpass << 'SCRIPT'
#!/bin/sh
echo "$GITHUB_TOKEN"
SCRIPT
  chmod +x .git-askpass

  # .envrc 생성
  cat > .envrc << EOF
export GITHUB_TOKEN="${GITHUB_TOKEN}"
export GIT_ASKPASS="$(pwd)/.git-askpass"
EOF

  # direnv가 있으면 allow
  if command -v direnv &>/dev/null; then
    direnv allow "$VAULT_PATH"
  fi
fi
```

**주의**: `.envrc`와 `.git-askpass`는 `.gitignore`에 포함되어 있어 커밋되지 않는다.

### 4. 심볼릭 링크 검증

vault의 심볼릭 링크가 올바르게 존재하는지 확인한다. 깨진 링크가 있으면 재생성한다:

```bash
cd "$VAULT_PATH"

# 핵심 심볼릭 링크 목록
declare -A LINKS=(
  ["claude.md"]="agents/agents.md"
  ["claude/claude.md"]="../agents/agents.md"
  ["claude/rules.md"]="../agents/rules.md"
  ["claude/tools.md"]="../agents/tools.md"
  ["claude/tools/obsidian-cli.md"]="../../agents/tools/obsidian-cli.md"
)

for link in "${!LINKS[@]}"; do
  target="${LINKS[$link]}"
  dir=$(dirname "$link")
  mkdir -p "$dir"
  if [ ! -L "$link" ] || [ "$(readlink "$link")" != "$target" ]; then
    ln -sf "$target" "$link"
    echo "🔗 $link → $target (재생성)"
  fi
done

# .claude/skills 심볼릭 링크
mkdir -p .claude/skills
for skill_dir in agents/skills/*/; do
  skill_name=$(basename "$skill_dir")
  if [ ! -L ".claude/skills/$skill_name" ]; then
    ln -s "../../../agents/skills/$skill_name" ".claude/skills/$skill_name"
    echo "🔗 .claude/skills/$skill_name → agents/skills/$skill_name (재생성)"
  fi
done
```

### 5. Obsidian CLI(notesmd-cli) 확인/설치

이 vault의 스킬은 `obsidian` 명령을 호출한다. 실제 도구는 **notesmd-cli**(구 Yakitrak `obsidian-cli`가 개명됨, tap도 `yakitrak/yakitrak`로 변경)이며, `obsidian`은 그 별칭(symlink)이다. npm 패키지 `obsidian-cli`(obsidianqa 테스트툴)가 같은 `obsidian` 이름을 점유하는 경우가 있어 충돌을 먼저 정리한다.

```bash
# 1) obsidian이 npm obsidianqa(cli.js)이면 obsidianqa로 보존하고 이름 비우기
OBS_PATH="$(command -v obsidian 2>/dev/null)"
if [ -n "$OBS_PATH" ] && readlink "$OBS_PATH" 2>/dev/null | grep -q "node_modules/obsidian-cli/cli.js"; then
  echo "⚠️ obsidian 이름을 obsidianqa(npm)가 점유 중 → 'obsidianqa'로 보존"
  ln -sf "../lib/node_modules/obsidian-cli/cli.js" "$(dirname "$OBS_PATH")/obsidianqa"
  rm -f "$OBS_PATH"
fi

# 2) notesmd-cli 동작 확인
if command -v obsidian &>/dev/null && obsidian --version 2>&1 | grep -qi notesmd; then
  echo "✅ obsidian = notesmd-cli 사용 가능"
fi
```

CLI가 없으면 AskUserQuestion으로 설치 여부를 묻는다:

```
AskUserQuestion:
  question: "Obsidian CLI(notesmd-cli)가 설치되어 있지 않습니다. 설치할까요?"
  header: "CLI 설치"
  options:
    - label: "설치 (Recommended)"
      description: "brew tap yakitrak/yakitrak && brew install yakitrak/yakitrak/notesmd-cli 로 설치하고 obsidian 별칭을 만듭니다."
    - label: "건너뛰기"
      description: "설치 없이 진행합니다. 나중에 직접 설치할 수 있습니다."
```

- 설치 선택 시:

```bash
if command -v brew &>/dev/null; then
  brew tap yakitrak/yakitrak && brew install yakitrak/yakitrak/notesmd-cli
  ln -sf notesmd-cli "$(brew --prefix)/bin/obsidian"
  echo "✅ notesmd-cli 설치 완료 (obsidian 별칭 생성)"
else
  echo "⚠️ Homebrew가 설치되어 있지 않습니다. brew 설치 후 다시 시도해주세요."
  echo "   Homebrew 설치: https://brew.sh"
fi
```

- 건너뛰기 선택 시: 설치 없이 다음 단계로 진행

설치/검증 후, notesmd-cli가 vault 이름 기준으로 동작하도록 vault를 등록한다(`idea`·`session-out` 스킬이 이 등록에 의존한다):

```bash
if command -v obsidian &>/dev/null && obsidian --version 2>&1 | grep -qi notesmd; then
  obsidian add-vault "$VAULT_PATH" --set-default 2>/dev/null && echo "✅ vault 등록: $(basename "$VAULT_PATH")"
fi
```

### 6. Obsidian Vault 등록 안내

vault를 Obsidian 앱에서 열도록 안내한다:

```
Obsidian 앱에서 이 vault를 열어주세요:
1. Obsidian 실행
2. "Open folder as vault" 선택
3. 경로: $VAULT_PATH
```

### 7. 완료 응답

모든 세팅이 끝나면 간결하게 요약한다:

```
✅ Second Brain 세팅 완료!
📂 경로: $VAULT_PATH
🔗 심볼릭 링크: 정상
🔑 Git 인증: 설정 완료
📝 Obsidian CLI: [설치됨/미설치]

다음 단계:
- Obsidian 앱에서 vault를 열어주세요
- /second-brain:idea 로 아이디어 캡처
- /second-brain:session-out 으로 세션 마무리
```

## $ARGUMENTS 처리

사용자가 `/init ~/my-vault` 처럼 경로를 인자로 전달하면, 해당 경로를 `VAULT_PATH`로 사용하고 경로 질문을 건너뛴다. `$ARGUMENTS`가 비어있으면 AskUserQuestion으로 경로를 입력받는다.
