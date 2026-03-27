---
name: wrap-up
description: |
  세션을 마무리하는 스킬. 대화 중 얻은 인사이트를 lesson-learns에 저장하고, git commit & push로 마무리한다.
  사용자가 "끝", "마무리", "wrap up", "세션 끝", "오늘은 여기까지", "저장하고 끝내자",
  "커밋하고 끝", "정리해줘" 등을 말하면 트리거한다.
  대화 종료 시점뿐 아니라, 중간에 "지금까지 배운거 정리해줘" 같은 요청에도 트리거한다.
---

# Wrap-up

세션을 마무리할 때 인사이트를 캡처하고 변경사항을 커밋하는 워크플로우.

## 실행 순서

### 1. 인사이트 추출

세션 대화를 돌아보고 기록할 만한 교훈을 추출한다. 다음을 중심으로 살핀다:

- 새로 알게 된 사실이나 패턴
- 삽질한 뒤 찾은 해결법
- 앞으로 작업 방식을 바꿀 만한 깨달음
- 의사결정의 근거가 된 정보

기록할 인사이트가 없는 세션(단순 질문, 짧은 대화 등)이라면 이 단계를 건너뛴다. 사용자에게 "기록할 만한 인사이트가 없는 것 같아서 바로 커밋으로 넘어갈게요"라고 알린다.

### 2. 저장 위치 결정

세션에서 특정 프로젝트를 다뤘다면 해당 프로젝트의 lesson-learns에 저장한다. 매칭되는 프로젝트가 없으면 common-context/lesson-learns/에 저장한다.

```
# 프로젝트 관련 세션
projects/{프로젝트명}/lesson-learns/YYYY-MM-DD-주제.md

# 범용 세션
common-context/lesson-learns/YYYY-MM-DD-주제.md
```

저장 위치에 lesson-learns/ 폴더가 아직 없으면 agents.md + claude.md 심볼릭 링크와 함께 생성한다(프로젝트 구조 규칙 참조).

### 3. 파일 작성

Obsidian CLI로 노트를 생성한다:

```bash
obsidian create name="YYYY-MM-DD-주제" path="저장경로/YYYY-MM-DD-주제.md" content="..." silent
```

파일 형식은 간결하게 유지한다:

```yaml
---
date: 2026-03-27
context: 세션에서 무엇을 했는지 한 줄 요약
tags: [관련태그]
---
```

본문은 배운 점을 3-5줄로 짧게 정리한다. 각 항목은 `- ` 로 시작하며, 향후 검색이 쉽도록 구체적인 키워드를 포함한다.

**예시:**

```markdown
---
date: 2026-03-27
context: second-brain vault 폴더 구조 설계
tags: [obsidian, 폴더구조, 에이전트지침]
---

- agents/ 폴더를 source of truth로 두고 claude/는 심볼릭 링크로 미러하면 유지보수가 편하다
- 이정표 역할의 문서는 내용을 직접 담지 않고 위키링크로 상세 문서를 가리키게 한다
- 스킬도 agents/skills/에 원본을 두고 .claude/skills/에서 심볼릭 링크로 참조한다
```

### 4. Git commit & push

모든 변경사항을 커밋하고 push한다:

```bash
git add -A
git commit -m "docs: lesson-learns from session (YYYY-MM-DD)"
git push
```

커밋 메시지는 `docs: ` 접두사를 사용하고, 필요하면 본문에 변경 내역을 간략히 적는다. lesson-learns 외에 다른 변경사항(새 파일, 구조 변경 등)이 있으면 커밋을 분리하는 것이 좋다:

```bash
# 1) 작업 내용 커밋
git add -A
git commit -m "feat: 설명"

# 2) lesson-learns 커밋
git add -A
git commit -m "docs: lesson-learns from session (YYYY-MM-DD)"

git push
```

### 5. 마무리 응답

커밋/push가 끝나면 간결하게 요약한다:

> 📝 lesson-learns 저장 → `저장경로/파일명.md`
> ✅ 커밋 & push 완료

길게 되풀이하지 않는다.
