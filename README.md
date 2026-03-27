# Second Brain Plugin

Obsidian 기반 Second Brain vault를 자동 세팅하고, 아이디어 캡처·세션 마무리 워크플로우를 제공하는 Claude Code 플러그인.

## 설치

### 로컬 테스트

```bash
claude --plugin-dir ./second-brain-plugin
```

### GitHub에서 설치

```bash
# 마켓플레이스 등록 후
/plugin marketplace add dongho108/second-brain-plugin
/plugin install second-brain-plugin
```

## 설정

설치 시 다음 값을 입력합니다:

| 설정 | 설명 | 필수 |
|---|---|---|
| `github_token` | GitHub Personal Access Token (repo 권한 필요) | ✅ |
| `vault_path` | Vault clone 경로 (기본값: `~/second-brain`) | |

## 스킬

| 스킬 | 명령 | 설명 |
|---|---|---|
| init | `/second-brain:init` | Vault clone + 환경 세팅 |
| idea | `/second-brain:idea` | 아이디어 빠르게 캡처 |
| wrap-up | `/second-brain:wrap-up` | 세션 마무리 + lesson-learns 저장 |

## 빠른 시작

```bash
# 1. 플러그인 설치 후 vault 세팅
/second-brain:init

# 2. 아이디어 캡처
/second-brain:idea 앱에서 AI 추천 기능을 넣으면 전환율이 올라갈 것 같다

# 3. 세션 마무리
/second-brain:wrap-up
```
