---
name: update
description: |
  Second Brain vault를 원격 저장소와 git pull --rebase로 동기화하는 스킬.
  로컬 변경사항은 stash로 보존하고, 충돌 시 사용자에게 해결 방식을 안내한다.
  다음 상황에서 트리거한다:
  - 명시적 요청: "업데이트", "최신화", "동기화", "sync", "pull", "update", "당겨와", "새로고침"
  - 다른 디바이스에서 작업하고 돌아왔을 때: "다른 데서 작업했는데", "아이패드에서 했는데", "폰에서 메모했어"
  - 세션 시작 전 최신 상태 확인: "최신 상태 맞아?", "변경된 거 있어?", "밀린 거 있나"
  - /idea, /wrap-up 전 충돌 예방 목적: "먼저 땡겨와", "작업 전에 동기화"
---

# Update

원격 저장소의 최신 변경사항을 로컬 vault로 가져온다.

## 실행 순서

### 1. Vault 경로 결정

`$ARGUMENTS`가 있으면 해당 경로를 `VAULT_PATH`로 사용하고 아래 질문을 건너뛴다.
없으면 현재 디렉토리가 git repo인지 확인(`[ -d .git ]`)하여 사용한다.
둘 다 아니면 AskUserQuestion으로 입력받는다:

```
AskUserQuestion:
  question: "어떤 vault를 최신화할까요?"
  header: "Vault 경로"
  options:
    - label: "~/second-brain (기본값)"
      description: "홈 디렉토리 아래 second-brain 폴더를 최신화합니다."
    - label: "직접 입력"
      description: "원하는 vault 경로를 직접 지정합니다."
```

### 2. 사전 검증

```bash
cd "$VAULT_PATH"

# git 설치 확인
if ! command -v git &>/dev/null; then
  echo "❌ git이 설치되어 있지 않습니다. brew install git 또는 xcode-select --install 로 설치해주세요."
  exit 1
fi

# .git 디렉토리 확인
if [ ! -d .git ]; then
  echo "❌ $VAULT_PATH 는 git 저장소가 아닙니다. /init 으로 먼저 세팅해주세요."
  exit 1
fi

# 원격 저장소 확인
if [ -z "$(git remote)" ]; then
  echo "❌ 원격 저장소가 설정되어 있지 않습니다. /init 으로 먼저 세팅해주세요."
  exit 1
fi

# 진행 중인 rebase 확인
if [ -d .git/rebase-merge ] || [ -d .git/rebase-apply ]; then
  echo "⚠️ 진행 중인 rebase가 있습니다."
  echo "  계속하려면: git rebase --continue"
  echo "  취소하려면: git rebase --abort"
  exit 1
fi

# detached HEAD 확인
if ! git symbolic-ref HEAD &>/dev/null; then
  echo "⚠️ detached HEAD 상태입니다. git checkout main 으로 복구해주세요."
  exit 1
fi
```

### 3. 로컬 변경사항 stash

커밋하지 않은 작업 내용을 보존하는 가장 안전한 방법이 stash다. 불필요한 커밋을 만들지 않는다.

```bash
LOCAL_CHANGES=$(git status --porcelain)
if [ -n "$LOCAL_CHANGES" ]; then
  echo "📦 로컬 변경사항을 임시 저장합니다..."
  git stash push -m "update-skill-auto-stash-$(date +%s)"
  STASHED=true
else
  STASHED=false
fi
```

### 4. git pull --rebase

노트 vault에서 merge commit은 히스토리 노이즈다. rebase로 선형 히스토리를 유지한다.

```bash
BEFORE_HEAD=$(git rev-parse HEAD)
git pull --rebase 2>&1
PULL_EXIT=$?
AFTER_HEAD=$(git rev-parse HEAD)
```

pull 실패 시 에러 유형을 구분한다:

- **네트워크 에러** (exit code != 0, 충돌 파일 없음): "네트워크 연결을 확인해주세요." 출력 후 stash가 있으면 `git stash pop`으로 복원하고 종료.
- **인증 실패** (출력에 "Authentication failed" 또는 "403" 포함): "인증이 만료되었습니다. /init 으로 토큰을 재설정해주세요." 출력 후 stash 복원하고 종료.
- **충돌** (충돌 파일 존재): 아래 충돌 해결 플로우로 진행.

### 5. 결과 처리

pull 성공 시:

```bash
if [ "$STASHED" = true ]; then
  echo "📦 임시 저장한 변경사항을 복원합니다..."
  git stash pop 2>&1
  STASH_POP_EXIT=$?
  # stash pop 충돌 시 → 충돌 해결 플로우로 진행
fi
```

충돌 없이 완료되면 6단계(완료 응답)로 이동한다.

### 충돌 해결 (공통)

rebase 충돌이든 stash pop 충돌이든 동일한 플로우를 사용한다.

```bash
CONFLICTS=$(git diff --name-only --diff-filter=U)
echo "⚠️ 충돌이 발생한 파일:"
echo "$CONFLICTS"
```

```
AskUserQuestion:
  question: "충돌을 어떻게 해결할까요?"
  header: "충돌 해결"
  options:
    - label: "원격 버전 사용 (Recommended)"
      description: "원격 저장소의 내용으로 덮어씁니다. 로컬 변경사항은 사라집니다."
    - label: "로컬 버전 유지"
      description: "로컬의 내용을 유지합니다. 원격 변경사항은 무시됩니다."
    - label: "직접 해결"
      description: "충돌 파일을 직접 편집합니다."
```

선택에 따른 처리:

```bash
# 원격 버전 사용
for f in $CONFLICTS; do git checkout --theirs "$f" && git add "$f"; done
git rebase --continue 2>/dev/null  # rebase 충돌인 경우
git checkout --theirs . 2>/dev/null && git reset 2>/dev/null  # stash pop 충돌인 경우

# 로컬 버전 유지
for f in $CONFLICTS; do git checkout --ours "$f" && git add "$f"; done
git rebase --continue 2>/dev/null
git checkout --ours . 2>/dev/null && git reset 2>/dev/null

# 직접 해결
echo "충돌 파일: $CONFLICTS"
echo "편집 후: git add <파일> && git rebase --continue"
# 스킬 실행 중단
```

### 6. 완료 응답

```bash
if [ "$BEFORE_HEAD" != "$AFTER_HEAD" ]; then
  NEW_COMMITS=$(git log --oneline "$BEFORE_HEAD".."$AFTER_HEAD" | wc -l | tr -d ' ')
fi
```

업데이트가 있으면:

```
✅ Second Brain 최신화 완료!
📂 경로: $VAULT_PATH
📥 새로운 커밋: N개
```

이미 최신이면:

```
✅ 이미 최신 상태입니다!
📂 경로: $VAULT_PATH
```
