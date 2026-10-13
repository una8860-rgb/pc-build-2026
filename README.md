# 조립 PC 견적 프로젝트 — 작업 기록 & 공부 노트

> 2026-10-13 · Claude Code와 함께 "새 컴퓨터 부품 최저가 검색 + 꿀조합 찾기"를 하면서 한 일을 정리했습니다.
> 결과 페이지: **https://una8860-rgb.github.io/pc-build-2026/**

---

## 1. 한눈에 보기

| 단계 | 한 일 | 결과 |
|---|---|---|
| ① 개발 환경 | `uv` 설치 → 파이썬 가상환경(`.venv`) 생성 | ✅ Python 3.14 venv |
| ② 에이전트 스킬 | k-skill에서 `naver-shopping-search`, `danawa-price-search` 설치 | ✅ 다나와 최저가 조회 성공 |
| ③ YouTube MCP | `ia-programming/youtube-mcp` 설치 + **버그 4개 수정** | ✅ 한국어 검색·자막 추출 성공 |
| ④ Firecrawl MCP | `firecrawl-mcp` 로컬 설치, API 키 연결 | ✅ 키 검증 (크레딧 1,018) |
| ⑤ Claude Code 연결 | `.mcp.json`에 MCP 서버 2개 등록, 키는 `.env`에 분리 | ⏳ Claude Code 재시작 필요 |
| ⑥ 시장 조사 | 유튜브 10월 견적 영상 자막 분석 + 다나와 부품 30여 개 가격 조회 | ✅ 견적 A/B/C 도출 |
| ⑦ 배포 | GitHub 저장소 생성 → GitHub Pages로 견적 페이지 공개 | ✅ 위 링크 |

전체 흐름:

```
[유튜브 견적 영상] --youtube-mcp(자막)--> 이번 달 추천 조합/시장 동향 파악
                                              │
[다나와] --danawa-price-search--> 부품별 쇼핑몰 최저가(배송비 포함) 조회
                                              │
                                   견적 A/B/C 비교 → index.html → GitHub Pages
```

---

## 2. 단계별 정리

### ① uv로 파이썬 가상환경 만들기

`uv`는 Rust로 만든 아주 빠른 파이썬 패키지/프로젝트 관리자입니다. (`pip` + `venv` + `pyenv` 역할을 한 번에)

```powershell
# uv 설치 (Windows 공식 설치 스크립트) → C:\Users\<사용자>\.local\bin 에 설치됨
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

uv venv                    # 현재 폴더에 .venv 생성
.venv\Scripts\activate     # (선택) 가상환경 활성화
uv sync                    # pyproject.toml + uv.lock 기준으로 의존성 설치
uv run python server.py    # 활성화 없이 .venv 안에서 바로 실행
```

**배운 점**
- 설치 직후에는 **새 터미널을 열어야 PATH가 반영**됩니다. 이미 떠 있는 프로그램(Claude Code 포함)은 옛 PATH를 그대로 씁니다.
- PowerShell에서 `activate`가 막히면 `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` 한 번 실행.
- `uv run` / `uv pip`는 `.venv`를 자동으로 찾기 때문에 **활성화하지 않아도 됩니다.**
- 프로젝트에 `.python-version`(예: `3.10`)이 있으면 `uv sync`가 그 버전 파이썬을 알아서 받아옵니다.

### ② 에이전트 스킬(Agent Skills) 설치

스킬 = "이런 작업은 이렇게 해라"는 설명서(`SKILL.md`) + 보조 스크립트 묶음. 에이전트가 필요할 때 읽어서 따라 합니다.

```powershell
npx --yes skills add NomaDamas/k-skill --list                       # 저장소에 있는 스킬 목록(131개) 보기
npx --yes skills add NomaDamas/k-skill --skill naver-shopping-search --skill danawa-price-search -y
```

- 설치 위치: `halla/.agents/skills/<스킬이름>/` (Claude Code에는 심볼릭 링크로 연결)
- 설치 화면에 **보안 위험 평가(Gen/Socket/Snyk)** 가 나오고, *"Review skills before use; they run with full agent permissions"* 라고 경고합니다 → **설치 후 `SKILL.md`와 스크립트를 직접 읽어보는 습관**이 중요합니다.
- 두 스킬 모두 "로그인·주문·결제 안 함, 읽기 전용"으로 확인 후 사용했습니다.

다나와 스킬 사용법 (부품 가격 조사에 사용):

```bash
# 1) 검색 → 상품 후보와 pcode(다나와 상품 번호) 얻기
npx -y @nomadamas/k-skill@0 exec danawa-price-search scripts/danawa_search.py -- search "RTX 5060 Ti 16GB" --limit 6
# 2) pcode로 쇼핑몰별 가격(배송비 포함 실구매가, 카드 할인, 결제 조건) 보기
npx -y @nomadamas/k-skill@0 exec danawa-price-search scripts/danawa_search.py -- offers 101725832 --limit 4
```

**배운 점**
- 다나와 검색 결과의 가격 ≠ 실제 최저가. `offers`로 보면 **최저가는 대부분 "현금(계좌이체) 결제 조건"** 이었습니다. 카드 결제면 몇 % 더 비쌉니다.
- 검색어가 너무 넓으면(`SSD 1TB M.2`) 0건이 나오기도 함 → 모델명(`990 EVO Plus 1TB`)으로 구체화.

### ③ YouTube MCP 서버 설치 & 버그 수정

**MCP(Model Context Protocol)** = AI 앱(Claude Code 등)에 외부 도구를 꽂는 표준 규격. 서버가 "도구 목록"을 알려주면 AI가 그 도구를 호출합니다.
`youtube-mcp`가 제공하는 도구: `search-youtube`, `get-transcript`, `store-video-info`, `search-transcripts`

```powershell
git clone --depth 1 https://github.com/ia-programming/youtube-mcp.git
cd youtube-mcp
uv sync
```

그대로는 한국어 용도로 쓸 수 없어서 4가지를 고쳤습니다. 전체 변경 내용: [`study/youtube-mcp-fixes.patch`](study/youtube-mcp-fixes.patch)

| # | 증상 | 원인 | 수정 |
|---|---|---|---|
| 1 | 자막을 아예 못 가져옴 (`ParseError: no element found`) | `youtube-transcript-api==0.6.3`이 오래돼서 YouTube 변경에 깨짐 | `>=1.2.4`로 올리고 `YouTubeTranscriptApi().list()/fetch()` 새 API로 직접 호출, **한국어 자막 우선** |
| 2 | 한국어 검색 결과·자막이 빈 문자열 | `clean_text()`가 `encode('ascii','ignore')`로 **ASCII 외 문자를 전부 삭제** | 출력 불가능한 제어문자만 제거하도록 변경 |
| 3 | 도구에서 오류가 나면 서버가 죽음 | 설치된 `mcp` 버전의 `McpError`는 `ErrorData` 객체만 받음 | `McpError(ErrorData(code=..., message=...))` |
| 4 | `GOOGLE_API_KEY` 없으면 서버 시작 자체가 실패 | 임베딩 클라이언트를 시작 시점에 생성 | **처음 쓸 때 생성(lazy)** → 검색·자막은 키 없이 동작. 임베딩 모델도 `embedding-001` → `gemini-embedding-001`(768차원) |

**배운 점**
- 오픈소스를 가져다 쓸 땐 **실제로 돌려서 테스트**해야 합니다. README대로 설치는 됐지만 핵심 기능(자막)이 동작하지 않았습니다.
- 테스트 방법: MCP 클라이언트로 서버를 stdio로 띄우고 `list_tools` → `call_tool`을 직접 호출 (Claude Code가 하는 것과 동일).
- 버전을 `==`로 고정(pin)하면 재현성은 좋지만, 외부 서비스(YouTube)가 바뀌면 깨질 수 있습니다.
- Windows에서는 로그 파일 인코딩을 `utf-8`로 명시해야 한글이 안전합니다.

### ④ Firecrawl MCP

Firecrawl = 웹페이지를 AI가 읽기 좋은 마크다운으로 긁어오는(scrape/crawl/search) 서비스.

```powershell
npm install --prefix mcp-servers/firecrawl firecrawl-mcp@3.29.1
```

- 키가 없으면 **keyless 모드**(scrape·search만, IP당 제한)로 동작합니다.
- 키 검증: `GET https://api.firecrawl.dev/v2/team/credit-usage` (Authorization: Bearer 키) → 남은 크레딧 확인.

### ⑤ Claude Code에 MCP 등록 & API 키 관리

프로젝트 폴더의 `.mcp.json`에 서버를 적으면 Claude Code가 시작할 때 자동으로 띄웁니다. 예시: [`study/mcp.json.example`](study/mcp.json.example)

```json
{
  "mcpServers": {
    "youtube": {
      "command": "C:\\Users\\admin\\.local\\bin\\uv.exe",
      "args": ["--directory", "C:\\...\\halla\\youtube-mcp", "run", "server.py"],
      "env": { "PYTHONIOENCODING": "utf-8" }
    },
    "firecrawl": {
      "command": "node",
      "args": ["--env-file=C:\\...\\halla\\.env", "C:\\...\\firecrawl-mcp\\dist\\index.js"]
    }
  }
}
```

**API 키는 코드/설정에 직접 쓰지 않고 `.env` 한 곳에** 모았습니다. 예시: [`study/env.example`](study/env.example)
- youtube-mcp: 파이썬 `load_dotenv()`가 상위 폴더까지 `.env`를 찾아 읽음
- firecrawl: Node 20.6+의 `node --env-file=.env` 옵션으로 읽음 (추가 패키지 불필요)

**배운 점**
- 방금 설치한 `uv`가 Claude Code의 PATH에 없을 수 있어서 **절대 경로**로 지정했습니다.
- `.env`는 **절대 GitHub에 올리지 않기** (이 저장소에도 예시 파일만 있음).
- 키를 채팅창에 붙여넣지 않기. 이번엔 브라우저에서 "복사" 버튼 → 클립보드 → `.env`에 바로 저장해서 키가 화면에 노출되지 않게 했습니다.
- `.mcp.json`을 바꾼 뒤엔 **Claude Code 재시작 + 서버 승인**이 필요합니다.

### ⑥ 시장 조사 → 견적 결론

1. `search-youtube`로 "2026년 10월 컴퓨터 견적", "영상편집 컴퓨터 견적" 검색 → 영상 10개 자막 수집
2. 공통 결론 추출
   - 9월부터 **그래픽카드 공급가 상승**, 메모리·SSD도 크게 오름
   - 프리미어 편집: 인텔 **코어 울트라5 250K Plus**(가성비)/**울트라7 270K Plus**(전문가) 추천
   - 편집·3D 그래픽카드 정석: **RTX 5060 Ti 16GB**, 램 최소 32GB, SSD 1~2TB
3. 다나와로 실제 가격 확인 → **DDR5 32GB 61~68만원 vs DDR4 32GB 28~36만원** 발견
4. 결론: 예산(150~200만원) 안에서 16GB 그래픽카드 + 32GB 램을 지키려면 **DDR4 플랫폼(i5-14400 + B760M D4)** 이 꿀조합

| 견적 | 구성 | 합계 |
|---|---|---:|
| **A (추천)** | i5-14400 · B760M DS3H D4 · DDR4 32GB · **RTX 5060 Ti 16GB** · 990 EVO Plus 1TB · PA120 SE · 650W · DAVEN D7 Mesh | 약 208만원 |
| B (예산 내) | A에서 그래픽카드만 RTX 5060 8GB | 약 177만원 |
| C (참고) | 울트라5 250K Plus · B860M · **DDR5** 32GB · RTX 5060 Ti 16GB | 약 250만원 |

호환성 체크 포인트: 쿨러 높이 ≤ 케이스 한도, 그래픽카드 길이 ≤ 케이스 한도, 보드 규격(M-ATX → ATX 케이스 OK), 파워 ≥ 그래픽카드 권장 용량.

### ⑦ GitHub Pages 배포

```bash
cd pc-build-page
git init -b main && git add index.html && git commit -m "Add PC build price research page"
gh repo create una8860-rgb/pc-build-2026 --public --source . --push
gh api -X POST repos/una8860-rgb/pc-build-2026/pages -f "source[branch]=main" -f "source[path]=/"
# 1분 정도 빌드 후 https://una8860-rgb.github.io/pc-build-2026/ 접속
```

**배운 점**
- 무료 GitHub Pages는 **public 저장소**가 필요 → 개인정보·키가 없는지 확인 후 올리기.
- `gh api .../pages/builds/latest`로 빌드 상태(`building` → `built`) 확인 가능.
- `.nojekyll` 파일이 있으면 Jekyll 변환 없이 파일을 그대로 서빙합니다.

---

## 3. 트러블슈팅 모음

| 문제 | 해결 |
|---|---|
| `uv` 명령을 못 찾음 | 설치 안 돼 있었음 → 공식 스크립트로 설치, 새 터미널 열기 |
| `skills add ... --naver-shopping-search` | 올바른 형식은 `--skill naver-shopping-search` (`--list`로 이름 먼저 확인) |
| 자막 `ParseError` | `youtube-transcript-api` 1.x로 업그레이드 |
| 한국어가 사라짐 | ASCII 강제 변환 코드 제거 |
| 서버 시작 시 `DefaultCredentialsError` | 임베딩 클라이언트 지연 생성 |
| 다나와 검색 0건 | 검색어를 모델명 수준으로 구체화 |
| 일부 상품 offers 0개 | 단종·품절 → 다른 제품으로 대체 (G.SKILL DDR4 → HIKSEMI 16GB×2) |

---

## 4. 용어 정리

- **가상환경(venv)**: 프로젝트마다 따로 쓰는 파이썬 패키지 공간. 프로젝트끼리 버전 충돌 방지.
- **MCP**: AI 앱 ↔ 외부 도구를 연결하는 표준 프로토콜. 서버는 stdio(표준입출력)나 HTTP로 통신.
- **Agent Skill**: 에이전트용 작업 설명서 + 스크립트. MCP가 "도구"라면 스킬은 "사용법/노하우".
- **임베딩(embedding)**: 문장을 숫자 벡터로 바꾼 것. 비슷한 의미끼리 가까워져서 의미 검색에 사용 (`search-transcripts`).
- **벡터 DB(LanceDB)**: 임베딩을 저장하고 가까운 벡터를 빠르게 찾는 DB.
- **pcode**: 다나와 상품 고유 번호. `https://prod.danawa.com/info/?pcode=번호`
- **퀵싱크(Quick Sync)**: 인텔 내장그래픽의 영상 인코딩/디코딩 가속. F 모델(예: 14400F)은 내장그래픽이 없어 사용 불가.
- **VRAM**: 그래픽카드 메모리. 3D 장면·고해상도 편집에서 부족하면 느려지거나 실패.

---

## 5. 남은 할 일

- [ ] Google AI Studio 약관 동의 → API 키 발급 → `.env`의 `GOOGLE_API_KEY` 채우기 (자막 저장/의미 검색용)
- [ ] Claude Code 재시작 → `youtube`, `firecrawl` MCP 서버 승인
- [ ] 구매 직전 다나와에서 가격 재확인 (가격 변동 큼), 한 쇼핑몰에 몰아서 배송비 절약
- [ ] 조립 후 메인보드 BIOS 최신 업데이트 (14세대 인텔 안정화 마이크로코드)

## 6. 더 공부해 볼 거리

- MCP 서버 직접 만들기: Python `mcp` SDK로 도구 1개짜리 서버 → `.mcp.json`에 등록해 보기
- `study/youtube-mcp-fixes.patch`를 읽고 `git apply`로 원본 저장소에 직접 적용해 보기
- 다나와 스킬의 `danawa_search.py` 코드 읽기 (BeautifulSoup으로 HTML 파싱하는 방법)
- 이 README의 견적을 Firecrawl로 커뮤니티(퀘이사존 등) 후기와 교차 검증해 보기

## 파일 구성

```
index.html                       # 견적 결과 페이지 (GitHub Pages)
README.md                        # 이 문서
study/youtube-mcp-fixes.patch    # youtube-mcp 버그 수정 diff
study/mcp.json.example           # Claude Code MCP 설정 예시
study/env.example                # API 키 파일 형식 (값은 비움)
```
