---
name: save
description: |
  Second Brain vault의 변경사항을 git commit & push하는 스킬.
  인사이트 추출이나 파일 생성 없이 순수하게 현재 변경된 문서를 원격에 올려두는 용도다.
  session-out과 달리 세션 마무리가 아니라 중간 저장 개념이다.
  다음 상황에서 트리거한다:
  - 명시적 요청: "저장", "세이브", "save", "커밋", "올려줘", "푸시", "push", "변경사항 저장"
  - 중간 저장: "여기까지 올려둬", "일단 저장", "작업 올려놔", "커밋해줘"
  - 단순 동기화: "원격에 올려", "깃에 올려", "백업해줘"
  session-out("끝", "마무리", "wrap up", "정리해줘")과는 구분된다. 세션을 끝내는 게 아니라 작업을 이어가면서 중간에 저장만 하고 싶을 때 이 스킬을 쓴다.
---

# Save

vault 변경사항을 커밋하고 push하는 가벼운 워크플로우. 인사이트 추출이나 lesson-learns 생성 없이 git 작업만 수행한다.

## 실행 순서

### 1. Vault 경로 결정

`$ARGUMENTS`가 경로처럼 보이면(`/` 또는 `~`로 시작) 해당 경로를 `VAULT_PATH`로 사용한다.
아니면 현재 디렉토리가 git repo인지 확인(`[ -d .git ]`)하여 사용한다.
둘 다 아니면 AskUserQuestion으로 입력받는다:

```
AskUserQuestion:
  question: "어떤 vault를 저장할까요?"
  header: "Vault 경로"
  options:
    - label: "~/second-brain (기본값)"
      description: "홈 디렉토리 아래 second-brain 폴더의 변경사항을 저장합니다."
    - label: "직접 입력"
      description: "원하는 vault 경로를 직접 지정합니다."
```

### 2. 사전 검증

```bash
cd "$VAULT_PATH"

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
```

### 3. 변경사항 확인

```bash
CHANGES=$(git status --porcelain)
if [ -z "$CHANGES" ]; then
  echo "✅ 변경사항이 없습니다. 이미 최신 상태입니다."
  # 스킬 종료
fi
```

변경사항이 있으면 `git diff --stat`으로 요약을 확인한다. 이 정보는 자동 커밋 메시지 생성에도 쓰인다.

### 4. 커밋 & 푸시

#### 커밋 메시지 결정

`$ARGUMENTS`에 경로가 아닌 텍스트가 있으면 그것을 커밋 메시지로 사용한다.

없으면 자동 생성한다. 형식:

```
vault: save N files (영향받은_폴더들)
```

예시:
- `vault: save 3 files (ideas, projects/my-app)`
- `vault: save 1 file (common-context/lesson-learns)`

영향받은 폴더는 변경된 파일 경로에서 첫 번째 또는 두 번째 디렉토리 수준까지 추출하고, 중복을 제거한다.

#### 실행

```bash
git add -A
git commit -m "$COMMIT_MSG"
git push
```

push 실패 시:

- 원격에 새 커밋이 있어서 실패한 경우: `/update`로 먼저 동기화한 뒤 다시 `/save`를 실행하라고 안내한다.
- 인증 실패: `/init`으로 토큰을 재설정하라고 안내한다.
- 네트워크 에러: 연결 확인을 요청한다.

### 5. 완료 응답

```
✅ 저장 완료!
📂 경로: $VAULT_PATH
💾 커밋: $COMMIT_MSG
```

길게 되풀이하지 않는다.
