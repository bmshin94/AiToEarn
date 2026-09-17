# AiToEarn 프로젝트 분석 정리 (한국어)

> 이 문서는 AiToEarn 저장소를 직접 열어보고 정리한 분석 노트입니다.
> 코드 기준일: 2026-09-17 / 기준 커밋: `1f7f895`

## 저장소 주소

| 구분 | 주소 |
|---|---|
| 이 저장소 (포크) | https://github.com/bmshin94/AiToEarn |
| 원본 저장소 (upstream) | https://github.com/yikart/AiToEarn |
| 공식 사이트 (국제판) | https://aitoearn.ai |
| 공식 사이트 (중국판) | https://aitoearn.cn |
| 라이선스 | MIT (Copyright (c) 2025 AiToEarn) |

---

## 1. 이게 뭐하는 프로젝트인가

**AI로 콘텐츠를 만들고, 여러 SNS에 한 번에 발행하고, 댓글 응대까지 자동화하는 오픈소스 플랫폼.**

공식 슬로건은 네 가지 축으로 구성됩니다.

| 축 | 설명 |
|---|---|
| **Create** | AI가 영상/이미지 생성 (Grok, Seedance, Nano Banana 등), 배치 생성 지원 |
| **Publish** | 14개 SNS 채널에 원클릭 발행 + 캘린더 예약 |
| **Engage** | 댓글 조회/작성, 좋아요·팔로우 자동화, AI 자동 답변 |
| **Monetize** | CPS/CPE/CPM 기반 콘텐츠 마켓플레이스 (⚠️ 아래 6번 참고 — 오픈소스에는 미포함) |

---

## 2. 폴더 구조와 기술 스택

```
AiToEarn/
├── project/
│   ├── aitoearn-backend/       # NestJS + Nx 모노레포 (pnpm)
│   │   ├── apps/aitoearn-server    # 채널 연동 / 발행 / API Key / MCP 서버
│   │   ├── apps/aitoearn-ai        # AI 에이전트 / 초안 생성
│   │   └── libs/                   # auth, queue, mongodb, redis, redlock,
│   │                               # ali-oss, aws-s3, channel-db, nest-mcp ...
│   ├── aitoearn-web/           # Next.js 15 + React + Tailwind (dev 포트 6061)
│   └── aitoearn-electron/      # Electron + React 데스크탑 앱
│       └── electron/plat/          # 샤오홍슈/더우인/콰이/빌리빌리/스핀하오 어댑터
├── docker-compose.yml          # 원클릭 배포 (컨테이너 7종)
├── demo/                       # 플랫폼 서명(signature) 테스트용 HTML
├── nginx/                      # 리버스 프록시 설정
└── README.md / README_EN.md / README_JA.md  # 3개국어 문서
```

**기술 스택:** NestJS · Nx · pnpm · MongoDB(레플리카셋) · Redis · RustFS(S3 호환) ·
Next.js 15 · React · Tailwind · Electron · Playwright(E2E) · Node 20.x

---

## 3. 설치 및 사용법

### 방법 A — 웹사이트 (설치 불필요)
https://aitoearn.ai 접속 → 회원가입.

### 방법 B — Docker 셀프호스팅 (권장)

```bash
git clone https://github.com/bmshin94/AiToEarn.git
cd AiToEarn
docker compose up -d
```

→ http://localhost:8080 접속

`docker-compose.yml`이 띄우는 컨테이너:

| 컨테이너 | 역할 |
|---|---|
| `mongodb` (+ `mongodb-rs-init`) | 데이터 저장, 레플리카셋 자동 구성 |
| `redis` | 캐시 및 작업 큐 |
| `rustfs` (+ `rustfs-init`) | S3 호환 파일 저장소 (영상/이미지) |
| `aitoearn-server` | 발행 / 채널 연동 / MCP API |
| `aitoearn-ai` | AI 에이전트 서버 |
| `aitoearn-web` | 웹 프론트엔드 |
| `nginx` | 8080 포트 통합 프록시 |

> 설치 후 UI의 **Configuration** 메뉴에서 **Server → Relay** 와 **AI → Relay** 를 설정해야
> SNS 발행이 동작합니다. Relay를 쓰면 각 플랫폼 개발자 등록 없이 공식 자격증명을 대여할 수 있습니다.
> 설정 저장 후 **Save and restart** 를 눌러야 서비스가 설정을 다시 읽습니다.

### 방법 C — 소스 실행 (개발 모드)

```bash
cd project/aitoearn-backend
pnpm install
cp apps/aitoearn-ai/config/config.yaml     apps/aitoearn-ai/config/local.config.yaml
cp apps/aitoearn-server/config/config.yaml apps/aitoearn-server/config/local.config.yaml
pnpm nx serve aitoearn-ai        # 터미널 1
pnpm nx serve aitoearn-server    # 터미널 2

cd ../aitoearn-web
pnpm install && pnpm run dev     # 터미널 3 (http://localhost:6061)
```

> `AGENTS.md` 규칙: 루트에서 install/build 금지. backend/web 각 디렉터리에서 `pnpm` 사용.

---

## 4. 플러그인인가, 스킬인가, MCP인가 → 셋 다

세 가지가 서로 다른 층위로 모두 들어 있습니다.

### ① MCP 서버 — 사용자가 Claude/Cursor에 붙여 쓰는 창구

`libs/nest-mcp`에 NestJS용 MCP 프레임워크를 자체 구현했고
(`@Tool()` 데코레이터, Streamable HTTP + SSE 트랜스포트),
`apps/aitoearn-server/src/core/unified-mcp/unified-mcp.module.ts`에서 툴을 노출합니다.

노출되는 주요 툴:

| 툴 | 설명 |
|---|---|
| `createVideoDraft` | AI 영상 초안 생성 |
| `createImageTextDraft` | AI 이미지+텍스트 초안 생성 |
| `getDraftTaskStatus` | 생성 작업 상태 조회 |
| `createChannelPublishFlow` | 다채널 발행 플로우 생성 |
| `publishChannelTaskNow` | 즉시 발행 |
| `updateChannelPublishAt` | 예약 발행 시각 변경 |
| `cancelChannelPublishTask` | 발행 취소 |
| `listChannelPublishRecords` | 발행 기록 목록 |
| `listChannelPlatforms` | 플랫폼 메타/발행 제한/옵션 스키마 조회 |
| `listChannelWorks` / `getChannelWorkDetail` | 게시물 조회 |

### ② 플러그인 — OpenClaw 전용

```bash
npx -y @aitoearn/openclaw-plugin-cli
```

### ③ 스킬(SKILL.md) — 서버 내부 AI 에이전트가 사용

`apps/aitoearn-ai/src/core/agent/skills/` 에 13개:

`analyzing-videos`, `composing-videos`, `crawling-social-media`, `editing-images`,
`editing-videos`, `extracting-thumbnails`, `generating-drama-recaps`, `generating-images`,
`generating-videos`, `managing-content`, `removing-subtitles`, `transferring-video-styles`,
`translating-videos`

스킬이 다른 스킬을 호출하는 체이닝 구조입니다.
(예: `generating-videos` → 15초 초과 시 `editing-videos`를 로드해 클립 연결)

---

## 5. API Key

②③④ 방식은 모두 API Key가 필요합니다.

**발급:** 사이트 로그인 → 좌측 **Settings** → **API Key** → Create → 복사

**Claude Desktop 설정 예시** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "aitoearn": {
      "type": "http",
      "url": "https://aitoearn.ai/api/unified/mcp",
      "headers": { "x-api-key": "발급받은-키" }
    }
  }
}
```

**환경 매칭 주의 (401의 주범):**

| 키 발급처 | 사용해야 할 URL |
|---|---|
| `aitoearn.cn` (중국판) | `https://aitoearn.cn/api/unified/mcp` (SSE: `/api/unified/sse`) |
| `aitoearn.ai` (국제판) | `https://aitoearn.ai/api/unified/mcp` (SSE: `/api/unified/sse`) |

셀프호스팅이라면 도메인을 자체 주소(예: `localhost:8080`)로 바꾸면 됩니다.

**코드에서 확인한 구현** (`apps/aitoearn-server/src/core/api-key/api-key.service.ts`):

- 키 형식: `설정된 prefix` + `nanoid 48자리`
- DB에는 평문이 아닌 **해시만 저장** → 생성 시 1회만 노출, 분실 시 재발급 필요
- 검증 시마다 `lastUsedAt` 갱신

> 개선 제안: 현재 해시 알고리즘이 SHA-1입니다. 키 엔트로피가 높아 실질 위험은 낮지만,
> 포크해서 운영할 경우 SHA-256 이상으로 교체 권장.

---

## 6. ⚠️ 중요 — 오픈소스에 "없는" 기능

README에 크게 소개된 **Monetize(마켓플레이스/CPS·CPE·CPM 정산)** 기능은
**오픈소스 코드에 포함되어 있지 않습니다.** 공식 SaaS 서버에만 존재합니다.

검증 방법:

```bash
grep -rniE "\b(cps|cpe|cpm|marketplace|payout|wallet|withdraw|settle)\b" apps libs
# → 0건
```

`apps/aitoearn-server/src/core/` 의 실제 모듈 목록:
`api-key`, `assets`, `channels`, `content`, `publish-record`, `short-link`, `unified-mcp`, `user`

즉 셀프호스팅 버전은 **"돈 버는 도구"이지 "돈을 주는 플랫폼"이 아닙니다.**
정산 기능이 필요하면 직접 구현해야 합니다.

### 반대로, 실제로 들어있는 것 (확인 완료)

| 기능 | 위치 / 구현 |
|---|---|
| 14개 플랫폼 발행 | `channels/platforms/` — bilibili, douyin, facebook, **google-business**, instagram, kwai, linkedin, pinterest, rednote, threads, tiktok, twitter, wechat, youtube |
| 댓글 조회/작성 | `channels/engagement/` — `listComments`, `createComment` |
| 좋아요/팔로우 | `engagement.service.ts` — `callFunction` (like / account action) |
| 성과 분석 | `channels/analytics/` — `fetchAccountAnalytics`, `fetchWorkAnalytics`, **스냅샷 시계열 저장** |
| 단축링크 + 리다이렉트 | `short-link/` — 클릭 추적 기반 |
| AI 에이전트 | `apps/aitoearn-ai/` — 스킬 13개 |
| MCP 서버 | `libs/nest-mcp` + `unified-mcp` |

> 한국 플랫폼(네이버, 카카오, 티스토리 등) 어댑터는 **하나도 없습니다.** → 국내 시장의 빈틈.

---

## 7. 왜 GitHub에서 주목받는가

1. **문제가 현실적** — 영상 하나를 채널 10개에 올리는 고통은 크리에이터 공통 문제
2. **오픈소스인데 완성도가 높음** — 보통 유료 SaaS 영역인데 MIT로 전체 공개 + 도커 원클릭
3. **중국 플랫폼 연동이 독보적** — 샤오홍슈/더우인/콰이/빌리빌리/위챗은 구현 난도가 매우 높음
   (`demo/xhs/signature.js` 같은 서명 알고리즘 코드까지 포함)
4. **타이밍** — MCP / AI 에이전트 트렌드 정중앙에서 "AI + 수익화" 조합으로 등장
5. **문서화 수준** — 중/영/일 3개국어 동기화 (`AGENTS.md`에 규칙으로 명문화), Trendshift 등재
6. **네이밍** — "AiToEarn"이라는 이름 자체가 클릭을 유도

---

## 8. 로컬 AI 에이전트 구축에 참고할 가치

**매우 높음.** `apps/aitoearn-ai/package.json`에서 확인:

```json
"@anthropic-ai/claude-agent-sdk": "0.2.33",
"@anthropic-ai/sdk": "0.71.2",
"@musistudio/claude-code-router": "2.0.0"
```

즉 **Claude Agent SDK로 만든 실제 프로덕션 서비스**이며, 다음 패턴을 그대로 참고할 수 있습니다.

| 배울 것 | 위치 |
|---|---|
| 에이전트 실행 엔진 (`query()`, `AbortController`, 세션 유지, 프로세스 관리) | `agent/services/agent-runtime.service.ts` |
| 인프로세스 MCP 서버 (`createSdkMcpServer`) — 별도 서버 없이 코드 내 툴 정의 | 동일 파일 |
| 스킬 시스템 + 자동 초기화 | `agent/skills/*/SKILL.md`, `agent/skill-init.service.ts` |
| 외부 스킬을 해시로 잠그는 패키지매니저식 관리 | `skills-lock.json` |
| 실시간 스트리밍 (RxJS + SSE + KeepAlive 청크) | `agent-runtime.service.ts` |
| 중복 실행 방지 분산 락 | `libs/redlock` (`@yikart/redlock`) |
| 모델 라우팅 / 다중 프로바이더 | `agent/claude-code-router/` |
| 에이전트 작업 영속화 및 재개 | `ContentGenerationTaskRepository` |
| 도메인 MCP 툴 분리 | `agent/mcp/` (media, subtitle, image-edit, video-utils, volcengine/*) |

---

## 9. React / PHP로 만들 수 있는가

### React → 이미 React 기반

- `project/aitoearn-web` = Next.js 15 + React + Tailwind
  (`src/app/[lng]/` 아래 accounts, ai-social, chat, draft-box, tasks-history, agent-assets 등)
- `project/aitoearn-electron` = Electron + React

→ 백엔드는 그대로 두고 **프론트만 한국형으로 교체**하는 것이 가장 현실적인 커스터마이징 경로.

### PHP → 부분적으로 가능, 전면 이식은 비권장

| 가능 | 어려움 |
|---|---|
| 회원/인증/API Key 관리 | **Claude Agent SDK는 Node/TS 전용 — PHP 버전 없음** |
| SNS OAuth 연동 (Laravel Socialite) | MCP 서버 구현 (PHP SDK 빈약) |
| 예약 발행 스케줄러 (Scheduler/Horizon) | 장시간 스트리밍 응답 |
| 콘텐츠 CRUD, 대시보드 | 영상 처리 큐 (Node/Go가 유리) |

**권장 하이브리드 구조:**

```
[Laravel (PHP)]  회원 / 결제 / 대시보드 / 예약 / 발행관리
       │ HTTP
       ▼
[Node.js 에이전트 서버]  Claude Agent SDK + 스킬 + MCP
```

**전략 우선순위**

| 순위 | 전략 | 난이도 |
|---|---|---|
| 1 | 그대로 쓰고 React 프론트만 커스터마이징 | 낮음 |
| 2 | 한국 플랫폼 어댑터만 추가 (`channels/platforms/`) | 중간 |
| 3 | PHP + Node 하이브리드 신규 제작 | 높음 |
| 4 | 전면 PHP 재작성 | 매우 높음 (비권장) |

---

## 10. 수익화 아이디어

> 아래 금액은 README 기재 스폰서 단가와 일반적인 VPS 요금을 바탕으로 한 **추정치**입니다.
> 실제 계약 전 최신 단가를 반드시 재확인하세요.

### 1위 — 자영업 SNS 대행 (결과 판매형) ⭐ 최우선 추천

- **타겟:** 미용실, 헬스장, 필라테스, 카페, 치과, 학원 등 동네 사업자
- **가격안:** 베이직 29만원 / 스탠다드 49만원 / 프로 89만원 (월)
- **고객 1명당 월 현금 원가:** 약 1만원 미만 (AI 생성비 + 서버 분담)
- **필요 개발:** 거의 없음 — 도커로 띄우고 바로 시작 가능
- **무기:** `analytics` 스냅샷 시계열 → 월간 성과 리포트 자동화 = 계약 유지율 방어
- **리스크:** 낮음 (자기 계정에 자기 콘텐츠 발행)

**유닛 이코노믹스 (평균 40만원 기준)**

| 고객 수 | 매출 | 변동비 | 고정비 | 순이익 |
|---|---|---|---|---|
| 1명 | 40만원 | 1만원 | 5만원 | 약 34만원 |
| 3명 | 120만원 | 3만원 | 5만원 | 약 112만원 |
| 5명 | 200만원 | 5만원 | 6만원 | 약 189만원 |
| 10명 | 400만원 | 10만원 | 8만원 | 약 382만원 |

> 실제 병목은 비용이 아니라 **운영 시간**(고객당 월 4~5시간).
> 주 20시간 투입 시 고객 15~20명이 한계 → 검수 프로세스 시스템화가 확장의 열쇠.

### 2위 — 한국 플랫폼 어댑터 추가 → "한국형 AiToEarn"

- **근거:** 지원 플랫폼 14개 중 한국 서비스 0개
- **추가 순서:** 티스토리(Open API, 쉬움) → 카카오채널(비즈니스 API) → 네이버 블로그(⚠️ 아래)
- **구현 위치:** `apps/aitoearn-server/src/core/channels/platforms/<플랫폼명>/`
  (`platforms.registry.ts` 구조가 잘 잡혀 있어 기존 어댑터 복제 방식으로 접근 가능)
- **수익 모델:** 월 3~9만원 SaaS, 또는 대행사 화이트라벨 납품
- **⚠️ 경고:** 네이버 블로그는 공식 발행 API가 제한적입니다. 브라우저 자동화로 우회하면
  약관 위반 및 저품질 처분 위험 → 공식 정책 확인 후, 불가하면
  "AI 초안 생성 → 사용자가 직접 게시" 반자동 방식 권장

### 3위 — AI 콘텐츠 공장 (쇼츠 자동화)

- 활용 스킬: `generating-videos`, `generating-drama-recaps`, `translating-videos`,
  `extracting-thumbnails`, `composing-videos`
- `short-link` 모듈로 제휴링크 클릭 추적 → 성과 좋은 패턴만 대량 재생산
- **⚠️ 저작권 주의:** 드라마/영화 요약은 원저작권자 허락 없이는 위법.
  라이선스 확보 소스 또는 순수 AI 생성물만 사용할 것
- 유튜브 "반복적/저가치 콘텐츠" 정책으로 수익화 거절 가능성 존재

### 4위 — Monetize(정산) 모듈 직접 구현 → 플랫폼 사업

- 6번에서 확인했듯 오픈소스에는 정산 기능이 없음 → 직접 만들면 차별화
- 성과 측정 기반은 이미 존재: `fetchWorkAnalytics` + `saveWorkSnapshots` + `short-link`
- 수익: 거래액의 10~20% 수수료
- **⚠️ 규제:** 자금 중개는 전자금융거래법 이슈 가능 → 에스크로/PG 활용 설계 필요,
  **사업 개시 전 전문가 상담 필수**
- 닭-달걀 문제(광고주 ↔ 크리에이터) 및 개발량 3~6개월

### 5위 — MCP 래퍼 / "카톡으로 SNS 발행" 서비스

- AiToEarn을 백엔드 엔진으로 두고 한국형 UX만 자체 제작
- 월 1~3만원 구독 또는 건당 과금
- React 역량으로 프론트 제작 가능 → 1위 모델의 셀프서비스 버전

---

## 11. 90일 실행 로드맵 (1위 모델 기준)

**1~2주차 — 검증**
- 도커로 띄워 본인 계정에 실제 발행 성공시키기
- 릴스/쇼츠, 예약 발행, 댓글 조회 실제 동작 확인
- `analytics` 데이터가 실제로 쌓이는지 확인
- 안 되는 기능 발견 시 사업계획 즉시 수정

**3~4주차 — 첫 고객 (무료 베타 2명)**
- 지인 사업자 대상 1개월 무료 제공
- 목적은 매출이 아니라 운영 리스크 발견
- 발행 전 검수 워크플로 확립

**5~8주차 — 상품화**
- 무료 고객 유료 전환
- 월간 리포트 자동화 (`analytics` 활용)
- 랜딩페이지 제작, 계약서/이용약관 준비

**9~12주차 — 확장**
- 목표: 고객 5명 = 월 200만원
- 티스토리 또는 카카오채널 어댑터 추가로 차별화
- Before/After 성과 자료로 영업자료 제작

---

## 12. 법 / 약관 체크리스트

| # | 항목 | 내용 |
|---|---|---|
| 1 | 광고 표기 | 대행 콘텐츠에 `#광고`, `#협찬` 표기 (표시광고법) |
| 2 | 플랫폼 약관 | 발행 자동화는 대체로 허용 / **자동 좋아요·팔로우·대량 댓글은 금지인 곳 다수** |
| 3 | 계정 권한 | 고객 계정 위임 범위를 계약서에 명시 |
| 4 | 개인정보 | 고객 데이터 보관·파기 정책 수립 |
| 5 | 저작권 | 타인 콘텐츠 재가공 특히 위험, AI 생성물도 이슈 존재 |
| 6 | AI 표기 | 플랫폼별 AI 생성 콘텐츠 라벨링 정책 확인 |
| 7 | MIT 라이선스 | 재판매 가능. 단 저작권 표시 + 라이선스 전문 포함 필수 |
| 8 | 자금 중개 | 정산 플랫폼 운영 시 전자금융거래법 사전 검토 필수 |

> 특히 2번: Engage(자동 좋아요/팔로우)는 가장 화려해 보이지만 가장 위험합니다.
> 고객 계정 정지는 사업 중단으로 직결되므로 **"발행 + 분석"부터 시작**을 권장합니다.

---

## 13. 핵심 요약

| 질문 | 답 |
|---|---|
| 설치 | `docker compose up -d` → localhost:8080 |
| 플러그인/스킬/MCP | 셋 다 존재. 외부 사용자는 **MCP**로 연결 |
| API Key | 필수. `.cn` / `.ai` 환경과 키를 반드시 일치 (불일치 시 401) |
| 인기 이유 | 현실적 문제 + 높은 완성도 + 중국 플랫폼 독점 + 트렌드 타이밍 |
| 로컬 에이전트 참고 | 매우 높음 — Claude Agent SDK 프로덕션 레퍼런스 |
| 수익화 | 자영업 SNS 대행(즉시) → 한국 어댑터(차별화) → 플랫폼화(장기) |
| React/PHP | React는 이미 사용 중. PHP는 하이브리드 구조 권장 |

**가장 중요한 세 가지**

1. **Monetize 기능은 오픈소스에 없다** — 사업계획의 전제를 여기서부터 잡아야 함
2. **`analytics` 스냅샷이 숨은 자산** — 자동 리포트가 대행업의 계약 유지율을 좌우
3. **한국 플랫폼 부재가 기회** — 원본 팀은 국내 시장에 관심이 없으므로 선점 가능
