# 2026-08-04 · 로컬 실행 환경 구축 + 다중 AI 라우팅 실증

작성: HAA02 / 2026-08-04 / `release/v3.8.50`

---

## 목표

포크한 OmniRoute 를 이 머신에서 돌리고, "다양한 AI 를 쓸 수 있게" 프로바이더를 붙여
**실제로 추론 응답이 오는 것까지** 확인한다.

## 범위

**코드 변경 없음.** 소스는 `origin/release/v3.8.50` 과 동일하다
(`git rev-list --left-right --count origin/release/v3.8.50...HEAD` → `0 0`).
이번 커밋은 `memory/` 문서 추가뿐이다.

---

## 한 일 (순서대로)

### 1. 저장소 파악

토큰 절약을 위해 Haiku 서브에이전트 4개를 병렬로 풀어 ①설치 경로 ②프로바이더 레지스트리
③설정·키 배선 ④실행·검증 경로를 나눠 조사했다. 다만 **싼 모델의 사실 보고는 그대로 믿지 않고**
핵심 수치(Node 버전, 프로바이더 개수, 설치 상태)는 직접 재확인했다 — 실제로 Node 버전 보고가
머신 실측과 어긋났다.

확인된 구조:

- OpenAI 호환 게이트웨이. `src/app/api/v1/chat/completions/route.ts` 외 `/v1/models`,
  `/v1/embeddings`, `/v1/images/generations` 등
- 프로바이더 레지스트리 **206개** 디렉토리 (`open-sse/config/providers/registry/`),
  REGISTRY 는 `open-sse/config/providers/index.ts`
- 인터페이스 `RegistryEntry` / `RegistryModel` — `open-sse/config/providers/shared.ts`
- 라우팅: `open-sse/services/autoCombo/engine.ts`, `src/domain/policyEngine.ts`,
  `src/domain/fallbackPolicy.ts`
- 포트 20128, 헬스체크 `src/app/healthz/route.ts`

### 2. Node 런타임 (블로커였음)

시스템 Node v22.22.1 이 `engines: >=22.22.2 <23 || >=24.0.0 <27` 에 **패치 1 차이로 미달**.
nvm 이 없어 새로 설치 → Node **v24.19.0** + npm 11.17.0. → gotchas G-001, decisions D-001

### 3. `.env` + 시크릿

`.env.example`(129KB / 2,328줄) → `.env` 복사 후 필수 3종 생성:
`JWT_SECRET`(base64 48B), `API_KEY_SECRET`(hex 32B), `INITIAL_PASSWORD`(랜덤).
권한 600. `.gitignore:73` 의 `.env*` 로 추적 제외 확인.

> 🔐 값은 저장소에 기록하지 않았다. 로컬 `.env` 에만 있다.

### 4. 의존성 — 조용히 틀린 실패 2건

`npm install` 이 **exit 0 으로 성공**했지만 npm 11 이 의존성 install 스크립트를 차단해
optionalDependencies 트리가 통째로 빠졌다. 그 결과 `@huggingface/transformers` 가 없어
turbopack 이 컴파일에 실패하고 **모든 라우트가 500** 이 됐다.

`--allow-scripts` CLI 플래그는 `EALLOWSCRIPTS` 로 거부돼서, 추적 파일을 건드리지 않는
`npm install --include=optional --ignore-scripts` 로 해결했다.
→ gotchas G-002 · G-003 · G-004, decisions D-002

중간에 백그라운드 래퍼가 `exit code 0` 을 보고했지만 실제 npm 은 `EXIT=1` 이었다.
**래퍼 종료코드가 아니라 로그의 실제 종료코드를 봐야 한다.**

### 5. 기동 + 프로바이더 연결

모듈 설치 후에도 dev 서버가 실패한 해석을 캐시하고 있어 **재기동 + `.next/cache` 삭제**가 필요했다.

헤드리스로 프로바이더를 붙였다:
`POST /api/auth/login` → `GET /api/auth/csrf` → `POST /api/providers` (`x-csrf-token` 헤더 필요).
스키마는 `src/shared/validation/schemas/provider.ts:55`.

연결 성공 6개: `pollinations`, `hackclub`, `auggie`, `mimocode`, `g4f-pollinations`, `g4f-groq`
거부됨(`"Invalid provider"`): `duckduckgo-web`, `chipotle`, `theoldllm`, `felo-web`,
`veoaifree-web` — **원인 미규명**.

---

## 검증 결과 (실측)

| 게이트                                                   | 결과                                                                   |
| -------------------------------------------------------- | ---------------------------------------------------------------------- |
| `/healthz`                                               | HTTP 200 `ok`                                                          |
| `/` → `/dashboard` → `/login`                            | HTTP 200 (641KB)                                                       |
| 클라이언트 JS 청크                                       | **38/38 HTTP 200, 실패 0**                                             |
| 대시보드 로그인                                          | HTTP 200 `{"success":true}`                                            |
| `/v1/models`                                             | **673개** 모델                                                         |
| `/v1/chat/completions` (`g4f-groq/llama-3.1-8b-instant`) | HTTP 200, `'OMNIROUTE_OK'` 정확 반환, usage 48토큰                     |
| `auto/chat` · `auto`                                     | HTTP 200, `'AUTO_OK'` — 업스트림 `big-pickle` 자동 선택                |
| `auto/best-free`                                         | HTTP 200 — 업스트림 `felo-chat` 자동 선택                              |
| 서버 재기동 후 상태 유지                                 | 프로바이더 6개 SQLite 에서 복원, `/v1` 재호출 `'STILL_OK'`             |
| 정상 종료                                                | `[DB] SQLite WAL checkpoint completed (TRUNCATE).` → `[Shutdown] Bye.` |

**문서가 무료라고 안내한 프로바이더는 대부분 이미 키를 요구한다** — Pollinations 401,
HackClub 401, g4f-pollinations 402. 키 없이 실제로 응답이 온 건 `g4f-groq` 뿐이다.
→ gotchas G-005

---

## 미완 / 확인 안 한 것

추측으로 채우지 않고 그대로 남긴다.

- **테스트 스위트 미실행** — `npm run test:unit` 을 한 번도 돌리지 않았다. 회귀 기준선이 없다.
  이어받는 사람이 가장 먼저 하면 좋을 일.
- **프로덕션 빌드 미검증** — `npm run build` 미실행. dev(turbopack) 경로만 봤다.
- **Docker 경로 미검증** — `docker compose --profile base up -d` 미시도.
- **`onnxruntime-node` 네이티브 바이너리 없음** — 로컬 임베딩(의미 메모리) 기능은 런타임에
  실패할 수 있다. 라우팅·추론에는 영향 없음을 확인했다. → decisions D-002 트레이드오프
- **일부 프로바이더가 `"Invalid provider"` 로 거부되는 원인** — 웹 세션(쿠키) 계열은
  `/api/providers/bulk-web-session` 등 별도 플로우로 보이나 확인하지 않았다.

---

## 다음 작업 (가치 × 리스크 순)

1. **실사용 프로바이더 키 등록** — 대시보드 → Providers → Add Provider.
   OpenAI / Anthropic / Gemini / Groq 등. 무료 경로는 신뢰할 수 없다(G-005).
2. **회귀 기준선 확보** — `npm run test:unit`. 명령 정의는 `package.json` 의 `test:unit`
   (Node 내장 `--test` + tsx/esm). 통과 수를 기록해 두면 이후 변경의 기준선이 된다.
3. **프로덕션 빌드 검증** — `npm run build` → `npm start`. dev 에서만 확인한 상태다.
4. **거부되는 프로바이더 원인 규명** — `POST /api/providers` 의 provider 허용 목록이
   `src/shared/constants/providers` 쪽에 따로 있는지 확인부터.
5. **upstream 동기화 정책 결정** — 이 포크를 계속 upstream 과 맞출지, 독자 유지할지.
   결정에 따라 decisions D-003 / D-004 를 재검토한다.

---

## 로컬 상태 (git 밖)

| 항목         | 위치                                     | 비고                           |
| ------------ | ---------------------------------------- | ------------------------------ |
| `.env`       | 저장소 루트                              | 시크릿 3종. 추적 제외          |
| SQLite       | `~/.omniroute/storage.sqlite` (약 1.9MB) | 프로바이더 6개 + 발급한 API 키 |
| dev 로그     | `logs/omniroute-dev.log`                 | `.gitignore:227`               |
| node_modules | 4.3G                                     |                                |
| Node 24      | `~/.nvm/versions/node/v24.19.0/`         | nvm default alias              |

세션 종료 시 서버는 정상 종료해 뒀다(WAL 체크포인트 완료, 포트 20128/20131/20132 해제).
