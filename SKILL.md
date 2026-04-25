# Lightbulb Skill

AI 에이전트가 웹/디자인 작업 시 창의성을 높이기 위해 참조하는 영감/인사이트 저장소.

## 데이터 구조

`ideas.json` 단일 파일로 관리.

```json
{
  "id": "lb-NNN",
  "title": "인사이트 제목",
  "body": "내용 (3~5문장 영감형 서술)",
  "category": "web | design | ux | mobile",
  "tags": ["태그1", "태그2"],
  "date": "YYYY-MM-DD",
  "source": "ceo-memo | agent-discovery | external-link | gemini-discussion",
  "added_by": "에이전트명",
  "status": "raw | reviewed | in-use | archived",
  "url": "선택적 외부 링크"
}
```

### 카테고리 설명
- `web` — 인터랙션/애니메이션/WebGL/CSS 창의적 기법
- `design` — 미술 조형원리, 편집 디자인, 광고 연출, 색채 철학
- `ux` — UX 심리학, 브랜딩 철학, 전환 설계
- `mobile` — 모바일 특화 UX/인터랙션

---

## 에이전트 사용법

### 1. 전체 랜덤 5~6개 호출 (기본 사용법)
작업 시작 전 전 카테고리에서 랜덤으로 5~6개 인사이트를 가져와 영감으로 활용:

```bash
curl -s https://raw.githubusercontent.com/hw5511/lightbulb-skill/main/ideas.json | python -c "
import sys, json, random
data = json.load(sys.stdin)
ideas = [i for i in data['ideas'] if i['status'] in ['reviewed', 'in-use']]
sample = random.sample(ideas, min(6, len(ideas)))
for i in sample:
    print(f'[{i[\"category\"]}] {i[\"title\"]}')
    print(f'  {i[\"body\"]}')
    print()
"
```

### 2. 카테고리 필터링 + 랜덤 샘플링
특정 카테고리 인사이트만 집중적으로 참조할 때:

```bash
curl -s https://raw.githubusercontent.com/hw5511/lightbulb-skill/main/ideas.json | python -c "
import sys, json, random
data = json.load(sys.stdin)
ideas = data['ideas']

# 카테고리: web | design | ux | mobile
category = 'design'
filtered = [i for i in ideas if i['category'] == category and i['status'] in ['reviewed', 'in-use']]

pool = filtered if filtered else ideas
sample = random.sample(pool, min(5, len(pool)))
for i in sample:
    print(f'[{i[\"title\"]}]')
    print(f'{i[\"body\"]}')
    print()
"
```

### 3. 태그 기반 필터링
특정 기법/주제의 인사이트만 선택할 때:

```bash
curl -s https://raw.githubusercontent.com/hw5511/lightbulb-skill/main/ideas.json | python -c "
import sys, json, random
data = json.load(sys.stdin)
ideas = data['ideas']

# 원하는 태그로 변경 (예: scroll, typography, color, gsap, webgl, 광고, 조형원리 등)
tag = 'scroll'
filtered = [i for i in ideas if tag in i.get('tags', []) and i['status'] in ['reviewed', 'in-use']]

pool = filtered if filtered else ideas
sample = random.sample(pool, min(5, len(pool)))
for i in sample:
    print(f'[{i[\"category\"]}] {i[\"title\"]}')
    print(f'{i[\"body\"]}')
    print()
"
```

### 4. 다중 카테고리 혼합 호출
web + design 인사이트를 섞어서 균형있게 가져올 때:

```bash
curl -s https://raw.githubusercontent.com/hw5511/lightbulb-skill/main/ideas.json | python -c "
import sys, json, random
data = json.load(sys.stdin)
ideas = data['ideas']

categories = ['web', 'design']
filtered = [i for i in ideas if i['category'] in categories and i['status'] in ['reviewed', 'in-use']]
sample = random.sample(filtered, min(6, len(filtered)))
for i in sample:
    print(f'[{i[\"category\"]}] {i[\"title\"]}')
    print(f'{i[\"body\"]}')
    print()
"
```

### 5. 프롬프트 주입 패턴
claude -p 실행 전 인사이트를 읽어 프롬프트 상단에 주입한다:

```bash
INSIGHTS=$(curl -s https://raw.githubusercontent.com/hw5511/lightbulb-skill/main/ideas.json | python -c "
import sys, json, random
data = json.load(sys.stdin)
ideas = [i for i in data['ideas'] if i['status'] in ['reviewed','in-use']]
sample = random.sample(ideas, min(6, len(ideas)))
print('[LIGHTBULB INSIGHTS - 아래 인사이트들을 이번 작업의 영감으로 활용하라]')
for i in sample:
    print(f'- [{i[\"category\"]}] {i[\"title\"]}: {i[\"body\"]}')
")

claude --dangerously-skip-permissions --model sonnet "${INSIGHTS}

${USER_PROMPT}"
```

### 6. 아이디어 추가 (GitHub API)
```bash
# 현재 파일 SHA 확인
SHA=$(curl -s https://api.github.com/repos/hw5511/lightbulb-skill/contents/ideas.json \
  -H "Authorization: token ${LIGHTBULB_PAT}" | python -c "import sys,json; print(json.load(sys.stdin)['sha'])")

# 수정된 ideas.json을 base64로 인코딩하여 업로드
CONTENT=$(python -c "import base64; print(base64.b64encode(open('ideas.json','rb').read()).decode())")

curl -s -X PUT https://api.github.com/repos/hw5511/lightbulb-skill/contents/ideas.json \
  -H "Authorization: token ${LIGHTBULB_PAT}" \
  -H "Content-Type: application/json" \
  -d "{\"message\":\"Add new idea\",\"content\":\"${CONTENT}\",\"sha\":\"${SHA}\"}"
```

---

## PAT 설정

Fine-grained PAT (lightbulb-skill repo only, Contents: read+write) 발급 후:
- `C:\woohee_industries\00-본부\02-설정\bot_tokens.json` → `lightbulb_pat` 키로 저장
- 환경변수: `LIGHTBULB_PAT`

---

## 운영 규칙

- `status: reviewed` — 검증됨, 프롬프트 주입에 사용 가능 (기본)
- `status: in-use` — 현재 CLAUDE.md에 반영 중
- `status: raw` — 아직 검증 안 된 아이디어
- `status: archived` — 더 이상 사용 안 함
- 200건 초과 시 `index.json` 별도 생성 (카테고리/태그 매핑)
- 1MB 초과 시 `ideas-YYYY-MM.json`으로 분리
