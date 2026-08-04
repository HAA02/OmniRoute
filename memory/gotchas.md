# gotchas.md — 반복 함정과 교훈

> 같은 함정을 두 번 적지 않는다. 이미 있는 항목이면 새로 추가하지 말고 **기존 항목을 보강**한다.
> 모든 항목은 실제로 겪은 것만 적는다. 추측은 적지 않는다.

---

## G-001 · 시스템 Node 가 engines 요구치에 "패치 1" 차이로 미달일 수 있다

`package.json` 의 `engines` 는 `>=22.22.2 <23 || >=24.0.0 <27` 인데 머신의 Node 가 **v22.22.1**
이었다. 딱 패치 1 낮다. `.node-version` / `.nvmrc` 는 `24` 를 가리킨다.

`.npmrc` 에 `engine-strict` 가 없어서 **npm 이 설치를 막지 않는다.** 그래서 그냥 진행하면 나중에
빌드·런타임에서 원인 불명으로 깨질 수 있다.

**대응**: `nvm use 24` (또는 `nvm install 24`). 설치 전에 `node -v` 로 확인하는 습관을 들인다.

---

## G-002 · npm 11 은 의존성 install 스크립트를 차단한다 — 그런데 종료코드는 0 이다

`npm install` 이 **exit 0 으로 "성공"** 했는데 로그 끝에 `npm warn allow-scripts` 가 잔뜩 찍혔다.
npm 11 부터 의존성의 `install`/`postinstall` 스크립트가 기본 차단된다.

그 결과 `onnxruntime-node` 의 install 스크립트가 막혔고, 그걸 의존하는 **optionalDependencies 트리가
통째로 프루닝**됐다. 특히 `@huggingface/transformers@3.5.2` 가 설치되지 않았다.

이게 이 저장소에서 어떻게 터지는지가 핵심이다 → **G-003**.

**대응**: `npm install --include=optional --ignore-scripts` 로 optional 트리를 명시 설치한다.
설치 후 확인:

```bash
for p in @huggingface/transformers @atjsh/llmlingua-2 @tensorflow/tfjs better-sqlite3 \
         js-tiktoken keytar tls-client-node wreq-js; do
  [ -d "node_modules/$p" ] && echo "OK  $p" || echo "MISS $p"
done
```

> `--allow-scripts=<pkg>` **CLI 플래그는 쓸 수 없다** — 프로젝트 설치에서 `EALLOWSCRIPTS` 로 거부된다
> ("Add the entries to the `allowScripts` field in package.json, or to .npmrc, instead").
> 추적 파일을 고치기 싫다면 `--ignore-scripts` 가 답이다(→ decisions.md D-002).

---

## G-003 · `/healthz` 가 500 이면 healthz 문제가 아니라 **전역 컴파일 실패**다

`src/app/healthz/route.ts` 는 분기 하나짜리 파일이라 자체적으로 500 을 낼 수 없다. 그런데 500 이 났다.
dev 로그도 `next.js: 1259ms, application-code: 5ms` 로 애플리케이션 코드는 거의 시간을 안 썼다.

응답 HTML(Next dev 오류 페이지) 안에 진짜 원인이 있었다:

```
./src/lib/memory/embedding/transformersLocal.ts:32:32
Module not found: Can't resolve '@huggingface/transformers'
```

즉 **G-002 로 빠진 모듈 하나 때문에 turbopack 이 컴파일에 실패해 모든 라우트가 500** 이 된 것이다.
소스 주석은 "lazy import 라 안전"하다고 적혀 있지만(`await import()`), **turbopack 은 빌드 타임에
해석을 시도**하므로 lazy 여부와 무관하게 터진다.

**대응**

1. 500 이 나면 라우트를 의심하기 전에 **응답 본문에서 `Module not found` / `Failed to compile` 을 찾는다.**
   ```bash
   curl -s http://localhost:20128/healthz | grep -o 'Module not found[^"]*' | head -3
   ```
2. 모듈을 설치한 뒤에는 **dev 서버를 반드시 재기동**한다. 실패한 해석 결과가 캐시돼 있어
   설치만으로는 안 고쳐진다. `rm -rf .next/cache` 까지 하면 확실하다.

---

## G-004 · 래퍼의 "exit code 0" 을 믿지 말고 실제 로그를 본다

백그라운드 실행 래퍼가 `completed (exit code 0)` 으로 보고했지만, 로그 안의 실제 npm 종료코드는
`EXIT=1` (`npm error code EALLOWSCRIPTS`) 이었다. 래퍼는 자기 자신의 종료코드를 보고한 것이다.

**대응**: 명령을 `; echo "EXIT=$?"` 로 감싸 실제 종료코드를 로그에 남기고, 완료 알림이 아니라
**그 줄을 확인**한다. 이 저장소의 주된 실패 모드는 "종료코드 0 인데 결과가 틀린" 유형이다.

---

## G-005 · 문서가 안내하는 "무료 프로바이더"는 이미 키를 요구한다 (2026-08-04 기준)

`docs/getting-started/QUICK-START.md:44` 는 Kiro / OpenCode Free / Pollinations 를 "카드도 키도
필요 없음"으로 안내한다. 실제로 붙여 보면:

| 프로바이더         | 결과                                                          |
| ------------------ | ------------------------------------------------------------- |
| `pollinations`     | HTTP 401 `Authentication required. Please provide an API key` |
| `hackclub`         | HTTP 401                                                      |
| `g4f-pollinations` | HTTP 402 `Insufficient balance ... 0.0009 pollen`             |
| **`g4f-groq`**     | **HTTP 200 — 실제 응답 옴** ✅                                |

업스트림 서비스들의 무료 정책이 문서 작성 이후 바뀌었다. **문서를 근거로 "무료로 된다"고 판단하지 말고
직접 호출해 본다.**

또 `POST /api/providers` 는 레지스트리에 있는데도 일부 id 를 `"Invalid provider"` 로 거부한다
(`duckduckgo-web`, `chipotle`, `theoldllm`, `felo-web`, `veoaifree-web`). 웹 세션(쿠키) 계열은
별도 플로우(`/api/providers/bulk-web-session`)를 타는 것으로 보이나 **미확인**이다.

---

## G-006 · dev 서버를 에이전트/스크립트의 자식으로 띄우면 같이 죽는다

에이전트 잡의 백그라운드 프로세스로 `npm run dev` 를 띄웠더니, 잡이 정리될 때 서버도 함께 내려가
"localhost:20128 안 됨" 이 됐다. 서버·설정 문제가 아니었다.

**대응**: 독립 세션으로 띄운다.

```bash
setsid nohup npm run dev > logs/omniroute-dev.log 2>&1 < /dev/null &
```

확인: `ps -o pid,ppid,sid,cmd -p <pid>` 에서 **SID 가 자기 자신**이면 분리된 것이다.

> `pkill -f "npm run dev"` 는 **자기 자신의 셸 명령줄과도 일치**해 실행 중인 명령을 죽인다.
> `pkill -f "run-next.mjs dev"` 처럼 더 좁은 패턴을 쓴다.

---

## G-007 · 로그인 페이지의 HTTP 200 은 "브라우저에서 정상"을 뜻하지 않는다

`curl -L http://localhost:20128/` 는 `/dashboard` → `/login` 으로 307 두 번 후 200 을 준다.
그런데 서버 HTML 의 가시 텍스트는 `Loading...` 뿐이고 `<form>` 도 `type="password"` 도 없다.
폼은 **클라이언트 JS 가 그린다**.

그래서 curl 200 만으로는 화면이 뜬다고 말할 수 없다. 실제로 확인하려면 **JS 청크가 전부 200 인지**
봐야 한다(이 저장소 로그인 페이지는 39개 중 38개가 실제 요청 대상).

```bash
curl -s -L http://localhost:20128/ | grep -o '<script[^>]*src="[^"]*"' \
  | sed 's/.*src="//;s/"//' | while read s; do
    echo "$(curl -s -o /dev/null -w '%{http_code}' "http://localhost:20128$s")  $s"
  done | grep -v '^200' || echo "전부 200"
```

또한 **dev 모드 첫 접속은 turbopack 온디맨드 컴파일 때문에 수십 초 걸릴 수 있다.**
`Loading...` 에서 멈춘 것처럼 보여도 고장이 아니다.

---

## G-008 · 프로바이더 키는 `.env` 가 아니라 SQLite 에 있다

`.env.example` 이 2,328줄이라 "여기에 프로바이더 키를 다 넣는다"고 오해하기 쉽다. 실제로는
`*_API_KEY` 가 20개뿐이고 대부분 특수 용도(BIFROST, QDRANT, VISION_BRIDGE 등)다.

206개 프로바이더의 키는 **대시보드에서 입력 → `src/lib/db/apiKeys.ts` 가 AES-256-GCM 으로 암호화
→ `~/.omniroute/storage.sqlite`** 에 저장된다. 암호화 키가 `.env` 의 `API_KEY_SECRET` 이다.

**`API_KEY_SECRET` 을 재생성하면 기존 저장 키를 전부 복호화할 수 없게 된다.** 백업 없이 바꾸지 말 것.

---

## G-009 · `~/.omniroute/` 는 부팅이 알아서 만든다 — 기존 설치와 헷갈리지 말 것

첫 기동 로그에 `Migrated Electron secrets from ~/.omniroute/server.env to ~/.omniroute/.env` 가
찍혀서 "예전 설치가 있었나?" 싶었는데, 파일 mtime 을 보니 **방금 부팅이 만든 것**이었다
(로그는 UTC, 파일 mtime 은 KST 라 9시간 차이로 더 헷갈렸다).

**대응**: 기존 데이터 존재 여부는 로그 문구가 아니라 `ls -la ~/.omniroute/` 의 mtime 으로 판단하고,
**타임존 차이를 먼저 의심**한다.
