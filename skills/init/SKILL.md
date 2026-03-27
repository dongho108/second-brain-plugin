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

- `github_token`: 플러그인 설치 시 userConfig로 입력받은 GitHub PAT
  - 환경변수 `CLAUDE_PLUGIN_OPTION_GITHUB_TOKEN`으로 접근 가능
- `vault_path`: vault를 clone할 경로 (기본값: `~/second-brain`)
  - 환경변수 `CLAUDE_PLUGIN_OPTION_VAULT_PATH`로 접근 가능

## 실행 순서

### 1. 환경 확인

먼저 현재 환경을 점검한다:

```bash
# GitHub 토큰 확인
if [ -z "$CLAUDE_PLUGIN_OPTION_GITHUB_TOKEN" ]; then
  echo "ERROR: GitHub 토큰이 설정되지 않았습니다. 플러그인 설정에서 github_token을 입력해주세요."
  exit 1
fi

# vault 경로 결정
VAULT_PATH="${CLAUDE_PLUGIN_OPTION_VAULT_PATH:-$HOME/second-brain}"
```

### 2. Repo Clone

GitHub 토큰을 사용하여 repo를 clone한다:

```bash
VAULT_PATH="${CLAUDE_PLUGIN_OPTION_VAULT_PATH:-$HOME/second-brain}"
GITHUB_TOKEN="$CLAUDE_PLUGIN_OPTION_GITHUB_TOKEN"

# 이미 존재하면 pull
if [ -d "$VAULT_PATH/.git" ]; then
  echo "기존 vault 발견 — pull로 최신화합니다."
  cd "$VAULT_PATH"
  git pull
else
  # clone with token
  git clone "https://${GITHUB_TOKEN}@github.com/dongho108/second-brain.git" "$VAULT_PATH"
  cd "$VAULT_PATH"
fi
```

### 3. Git 인증 설정

`.envrc`와 `.git-askpass`를 생성하여 이후 push/pull이 자동으로 동작하도록 한다:

```bash
cd "$VAULT_PATH"

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

### 5. Obsidian CLI 확인

Obsidian CLI가 설치되어 있는지 확인한다:

```bash
if command -v obsidian &>/dev/null; then
  echo "✅ Obsidian CLI 사용 가능"
  obsidian help 2>/dev/null && echo "✅ Obsidian 앱 연결 확인"
else
  echo "⚠️ Obsidian CLI가 설치되어 있지 않습니다."
  echo "   설치: https://github.com/Yakitrak/obsidian-cli"
  echo "   brew install yakitrak/tap/obsidian-cli"
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
- /second-brain:wrap-up 으로 세션 마무리
```

## $ARGUMENTS 처리

사용자가 `/init ~/my-vault` 처럼 경로를 인자로 전달하면, 해당 경로를 `VAULT_PATH`로 사용한다. `$ARGUMENTS`가 비어있으면 `CLAUDE_PLUGIN_OPTION_VAULT_PATH` 또는 기본값 `~/second-brain`을 사용한다.
