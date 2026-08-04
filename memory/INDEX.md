# memory/INDEX.md — OmniRoute 작업 지식 진입점

> 이 저장소에서 작업을 이어받는다면 **여기부터** 읽는다.
> 현재 상태는 이 파일이 정본이다(항상 덮어씀). 세션별 이력은 `handoff/`에 쌓인다.

- 최신 인수인계: [handoff/2026-08-04-local-setup.md](handoff/2026-08-04-local-setup.md)
- 반복 함정: [gotchas.md](gotchas.md)
- 결정 로그: [decisions.md](decisions.md)

---

## 이 저장소가 무엇인가

`diegosouzapw/OmniRoute` 의 **포크**다. 206개 LLM 프로바이더를 하나의 OpenAI 호환
엔드포인트(`/v1`)로 라우팅하는 Next.js 게이트웨이 + CLI.

| 항목        | 값                                                     |
| ----------- | ------------------------------------------------------ |
| origin      | `https://github.com/HAA02/OmniRoute.git` (내 포크)     |
| upstream    | `https://github.com/diegosouzapw/OmniRoute.git` (원본) |
| 현재 브랜치 | `release/v3.8.50`                                      |
| 버전        | `omniroute` v3.8.50                                    |
| 기본 포트   | 20128 (대시보드 + `/v1` API)                           |

⚠️ `release/v3.8.50` 은 upstream 을 **그대로 미러링하던 브랜치**였다. 이 `memory/` 커밋이
포크의 첫 divergence 다. upstream 재동기화 시 이 점을 감안한다(→ decisions.md D-003).

---

## 현재 상태 (2026-08-04)

**로컬 실행 환경 구축 완료. 코드 변경은 없음.**

이번 세션은 기능 개발이 아니라 "이 프로젝트를 내 머신에서 돌리고 다양한 AI를 붙일 수 있게" 만드는
셋업이었다. 소스는 upstream 과 동일하다(`git rev-list --left-right --count origin/release/v3.8.50...HEAD`
→ `0 0`).

### 검증된 것 (실측)

| 게이트                                                   | 결과                                              |
| -------------------------------------------------------- | ------------------------------------------------- |
| `/healthz`                                               | HTTP 200 `ok`                                     |
| `/` → `/dashboard` → `/login`                            | HTTP 200 (641KB)                                  |
| 클라이언트 JS 청크                                       | 38/38 HTTP 200                                    |
| `/v1/models`                                             | 673개 모델 노출                                   |
| `/v1/chat/completions` (`g4f-groq/llama-3.1-8b-instant`) | HTTP 200, `OMNIROUTE_OK` 정확 반환 (usage 48토큰) |
| `auto/chat` 자동 라우팅                                  | HTTP 200, `AUTO_OK` — 업스트림 자동 선택 동작     |
| 대시보드 로그인                                          | HTTP 200 `{"success":true}`                       |
| 프로바이더 연결                                          | 6개 활성, 재기동 후에도 SQLite 에서 복원됨        |

### 아직 확인 안 된 것 (추측 금지 — 그대로 "미확인")

- **테스트 스위트 미실행** — `npm run test:unit` 을 한 번도 돌리지 않았다. 회귀 기준선 없음.
- **프로덕션 빌드 미검증** — `npm run build` 미실행. dev(turbopack) 경로만 확인했다.
- **Docker 경로 미검증** — `docker compose --profile base up -d` 는 시도하지 않았다.
- 일부 프로바이더(`duckduckgo-web`, `chipotle`, `theoldllm`, `felo-web`, `veoaifree-web`)는
  `POST /api/providers` 가 `"Invalid provider"` 로 거부한다. 레지스트리에는 존재하는데 왜
  거부되는지 **원인 미규명**.

---

## 실행 방법

```bash
cd <저장소 루트>
nvm use 24                     # 시스템 node 는 버전 미달 (→ gotchas G-001)
npm run dev                    # http://localhost:20128
```

터미널을 닫아도 유지하려면 독립 세션으로 띄운다(→ gotchas G-006):

```bash
setsid nohup npm run dev > logs/omniroute-dev.log 2>&1 < /dev/null &
```

정지: `pkill -f "run-next.mjs dev"` — 로그 끝에 `[Shutdown] SQLite database checkpointed and closed.`
가 찍히면 정상 종료다.

### 로컬 전용 파일 (git 추적 안 됨, 각자 만들어야 함)

| 경로                          | 내용                                          | 비고                    |
| ----------------------------- | --------------------------------------------- | ----------------------- |
| `.env`                        | `.env.example`(2,328줄) 복사 후 필수 3종 채움 | `.gitignore:73` `.env*` |
| `~/.omniroute/storage.sqlite` | 프로바이더 연결·API 키(암호화)                | 저장소 밖               |
| `logs/`                       | dev 서버 로그                                 | `.gitignore:227`        |

**필수 시크릿 3종** (`.env.example` 1절 "REQUIRED SECRETS"):

| 변수               | 생성법                    | 역할                                                  |
| ------------------ | ------------------------- | ----------------------------------------------------- |
| `JWT_SECRET`       | `openssl rand -base64 48` | 대시보드 세션 토큰 서명                               |
| `API_KEY_SECRET`   | `openssl rand -hex 32`    | SQLite 내 프로바이더 키 AES-256-GCM 암호화            |
| `INITIAL_PASSWORD` | 강한 랜덤값               | 최초 대시보드 로그인 (부팅 시 bcrypt 로 마이그레이션) |

> 🔐 실제 값은 이 저장소에 **절대 기록하지 않는다**. 로컬 `.env` 에만 둔다.
> `API_KEY_SECRET` 을 바꾸면 SQLite 에 저장된 프로바이더 키를 복호화할 수 없게 된다.

---

## 다양한 AI 붙이는 법 — 핵심 구조

**프로바이더 API 키는 `.env` 가 아니라 대시보드 → SQLite 에 암호화 저장된다.**
`.env.example` 이 2,328줄이라 착각하기 쉽지만 `*_API_KEY` 는 20개뿐이고 대부분 특수 용도다.

- 정본 레지스트리: `open-sse/config/providers/index.ts` (REGISTRY), 하위 디렉토리 **206개**
  (`open-sse/config/providers/registry/<id>/index.ts`)
- 프로바이더 인터페이스: `open-sse/config/providers/shared.ts` — `RegistryEntry` / `RegistryModel`
- 라우팅 진입점: `open-sse/services/autoCombo/engine.ts`(Auto-Combo),
  `src/domain/policyEngine.ts`, `src/domain/fallbackPolicy.ts`
- 연결 생성 API: `POST /api/providers` — 스키마는 `src/shared/validation/schemas/provider.ts:55`
  (`createProviderSchema`: `{provider, name, apiKey?, priority?, ...}`)

### 붙이는 경로 3가지

1. **대시보드** (권장): `http://localhost:20128` → Providers → Add Provider → 키 입력
2. **HTTP API** (헤드리스): `POST /api/auth/login` → `GET /api/auth/csrf` → `POST /api/providers`
   (변경 요청에는 `x-csrf-token` 헤더 필요)
3. **CLI**: `omniroute providers list|test|test-all`, `omniroute doctor`
   ⚠️ `omniroute providers available` 은 **6개짜리 큐레이션 목록**만 보여준다(openai, anthropic,
   google, openrouter, groq, mistral). 206개 전체가 아니다 — CLI 만 보고 "6개뿐"이라 판단하지 말 것.

### 키 없이 되는 프로바이더

`authType: "none" | "optional"` 인 항목은 16개지만, **대부분 업스트림이 이미 키를 요구한다**
(→ gotchas G-005). 2026-08-04 기준 키 없이 실제 응답이 온 건 `g4f-groq` 뿐이다.

---

## 클라이언트 연결

```
Base URL : http://localhost:20128/v1
API Key  : 대시보드 → API Keys 에서 발급 (OmniRoute 접근용, 업스트림 키와 무관)
Model    : auto   (또는 auto/chat, auto/best-free, <provider>/<model>)
```

`omniroute setup-claude | setup-cursor | setup-codex | setup-continue | ...` 로 각 도구 설정을
자동 생성할 수 있다(`bin/cli/commands/setup-*.mjs`).

---

## 참고 문서

| 경로                                       | 내용                                             |
| ------------------------------------------ | ------------------------------------------------ |
| `docs/getting-started/QUICK-START.md`      | 3분 시작 — ⚠️ 무료 프로바이더 안내는 낡음(G-005) |
| `docs/getting-started/PROVIDERS-GUIDE.md`  | 프로바이더 연결 단계별                           |
| `docs/getting-started/FREE-TIERS-GUIDE.md` | 무료 티어                                        |
| `docs/ENVIRONMENT.md`                      | 환경변수 전체 계약                               |
| `CLAUDE.md` (47KB) / `AGENTS.md` (37KB)    | upstream 의 코딩 규약·빌드 명령                  |
