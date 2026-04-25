# Lightbulb Skill

AI 에이전트가 웹/디자인/아이디어 작업 시 창의성을 높이기 위해 참조하는 인사이트 저장소.

## 데이터 구조

`ideas.json` 단일 파일로 관리.

```json
{
  "id": "lb-NNN",
  "title": "인사이트 제목",
  "body": "내용 (마크다운 가능)",
  "category": "web | design | ai | biz | workflow",
  "tags": ["태그1", "태그2"],
  "date": "YYYY-MM-DD",
  "source": "ceo-memo | agent-discovery | external-link | sticky",
  "added_by": "에이전트명",
  "status": "raw | reviewed | in-use | archived",
  "url": "선택적 외부 링크"
}
```

## 에이전트 사용법

### 1. 전체 아이디어 읽기
```bash
curl -s https://raw.githubusercontent.com/hw5511/lightbulb-skill/main/ideas.json
```

### 2. 카테고리 필터링 + 랜덤 샘플링 (python)
```bash
curl -s https://raw.githubusercontent.com/hw5511/lightbulb-skill/main/ideas.json | python -c "
import sys, json, random
data = json.load(sys.stdin)
ideas = data['ideas']

# 카테고리 필터 (원하는 카테고리로 변경)
category = 'web'
filtered = [i for i in ideas if i['category'] == category and i['status'] in ['reviewed', 'in-use']]

# 랜덤 3개 샘플링 (없으면 전체에서)
pool = filtered if filtered else ideas
sample = random.sample(pool, min(3, len(pool)))
for i in sample:
    print(f'- [{i[\"title\"]}] {i[\"body\"]}')
"
```

### 3. 프롬프트 주입 패턴
claude -p 실행 전 아이디어를 읽어 프롬프트 상단에 주입한다:

```bash
INSIGHTS=$(curl -s https://raw.githubusercontent.com/hw5511/lightbulb-skill/main/ideas.json | python -c "
import sys, json, random
data = json.load(sys.stdin)
ideas = [i for i in data['ideas'] if i['status'] in ['reviewed','in-use']]
sample = random.sample(ideas, min(5, len(ideas)))
print('[LIGHTBULB INSIGHTS - 이 인사이트들을 이번 작업에 반영하라]')
for i in sample:
    print(f'- {i[\"title\"]}: {i[\"body\"]}')
")

claude --dangerously-skip-permissions --model sonnet "${INSIGHTS}

${USER_PROMPT}"
```

### 4. 아이디어 추가 (GitHub API)
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

## PAT 설정

Fine-grained PAT (lightbulb-skill repo only, Contents: read+write) 발급 후:
- `C:\woohee_industries\00-본부\02-설정\bot_tokens.json` → `lightbulb_pat` 키로 저장
- 환경변수: `LIGHTBULB_PAT`

## 운영 규칙

- `status: raw` — 아직 검증 안 된 아이디어
- `status: reviewed` — 검증됨, 프롬프트 주입에 사용 가능
- `status: in-use` — 현재 CLAUDE.md에 반영 중
- `status: archived` — 더 이상 사용 안 함
- 1MB 초과 시 `ideas-YYYY-MM.json`으로 분리
- 200건 초과 시 `index.json` 별도 생성 (카테고리/태그 매핑)
