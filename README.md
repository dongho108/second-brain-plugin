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

`/second-brain:init` 실행 시 대화형으로 필요한 설정을 입력받습니다:

| 설정 | 설명 | 입력 방식 |
|---|---|---|
| `github_token` | GitHub Personal Access Token (repo 권한 필요) | 대화형 입력 (건너뛰기 가능) |
| `vault_path` | Vault clone 경로 (기본값: `~/second-brain`) | 대화형 선택 또는 `/init <경로>` 인자 |

## 스킬

| 스킬 | 명령 | 설명 |
|---|---|---|
| init | `/second-brain:init` | Vault clone + 환경 세팅 |
| idea | `/second-brain:idea` | 아이디어 빠르게 캡처 |
| session-out | `/second-brain:session-out` | 세션 마무리 + lesson-learns 저장 |

## 빠른 시작

```bash
# 1. 플러그인 설치 후 vault 세팅
/second-brain:init

# 2. 아이디어 캡처
/second-brain:idea 앱에서 AI 추천 기능을 넣으면 전환율이 올라갈 것 같다

# 3. 세션 마무리
/second-brain:session-out
```
