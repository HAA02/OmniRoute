# decisions.md — 결정 로그 (ADR-lite)

> 갈림길에서 무엇을 왜 골랐나. 나중에 "그때 왜 그렇게 했지"에 답하기 위한 기록.
> 형식: 결정 / 맥락 / 대안 / 근거 / 되돌리는 법.

---

## D-001 · Docker 대신 nvm + Node 24 소스 실행을 택한다

- **일자**: 2026-08-04 · **결정자**: 사용자 (선택지 제시 후 명시적 선택)
- **맥락**: 시스템 Node 가 v22.22.1 로 `engines` 요구치(`>=22.22.2`)에 패치 1 차이로 미달이었다
  (→ gotchas G-001). 머신에는 Docker 29.4.3 + Compose v5.1.3 이 이미 설치돼 있어서
  `docker compose --profile base up -d` 로 우회할 수도 있었다.
- **대안**
  1. Docker 기동 — 시스템 Node 를 안 건드림. 이미지가 node:26 을 씀. 단 소스 수정 → 즉시 반영되는
     개발 루프가 아님.
  2. **nvm + Node 24 (채택)** — 개발·디버깅에 적합, 핫리로드.
  3. 현재 Node 22.22.1 로 강행 — `engine-strict` 가 없어 설치는 되지만 실패 시 원인 추적이 어려움.
- **근거**: 목적이 "이 프로젝트를 셋업하고 다양한 AI 를 붙이는 것"이고, 프로바이더 설정을 만지며
  확인하는 작업이라 핫리로드가 있는 소스 실행이 맞다. 시스템 Node 는 그대로 두고 nvm 으로 격리했다.
- **결과**: Node v24.19.0 + npm 11.17.0. nvm default alias 24, `~/.bashrc` 에 배선됨.
- **되돌리기**: `nvm alias default system` 후 nvm 제거. 저장소에는 아무 흔적도 남지 않는다.

---

## D-002 · 추적 파일(`.npmrc` / `package.json`)을 고치지 않고 `--ignore-scripts` 로 우회한다

- **일자**: 2026-08-04 · **결정자**: Claude (판단 후 진행)
- **맥락**: npm 11 이 의존성 install 스크립트를 차단해 optionalDependencies 트리가 누락됐고,
  그 탓에 모든 라우트가 500 이 됐다(→ gotchas G-002, G-003).
- **대안**
  1. `.npmrc` 에 `allow-scripts` 항목 추가 — **`.npmrc` 는 git 추적 파일**이라 포크가 upstream 과
     불필요하게 갈라진다.
  2. `package.json` 에 `allowScripts` 필드 추가 — 같은 문제. 더 눈에 띈다.
  3. `--allow-scripts=<pkg>` CLI 플래그 — **불가**. 프로젝트 설치에서 `EALLOWSCRIPTS` 로 거부된다.
  4. **`npm install --include=optional --ignore-scripts` (채택)**
- **근거**: 이 저장소는 서드파티 OSS 포크다. upstream 과 계속 동기화할 것이라면 추적 파일 수정은
  매 동기화마다 충돌점이 된다. 필요한 건 `@huggingface/transformers` 가 **디스크에 존재해서
  turbopack 이 해석에 성공하는 것**뿐이고, 그건 JS 파일만 있으면 된다.
- **트레이드오프**: `onnxruntime-node` 의 네이티브 바이너리는 여전히 없다. 로컬 임베딩(의미 메모리)
  기능을 실제로 쓰면 런타임에 실패할 수 있다 — **미검증**. 라우팅·추론 경로에는 영향 없음을 확인했다.
- **되돌리기**: 네이티브 기능이 필요해지면 `.npmrc` 에 `allow-scripts` 를 추가하고 재설치한다.

---

## D-003 · `memory/` 를 `release/v3.8.50` 브랜치에 커밋한다 (포크의 첫 divergence)

- **일자**: 2026-08-04 · **결정자**: Claude (사용자의 "깃허브 푸시" 지시 하에)
- **맥락**: `release/v3.8.50` 은 커밋 시점까지 `origin` 과 완전 동기였고
  (`git rev-list --left-right --count` → `0 0`), 사실상 upstream 미러였다.
- **대안**
  1. 별도 브랜치(`docs/setup-memory`) 생성 — 릴리스 브랜치를 깨끗하게 유지.
  2. **현재 브랜치에 커밋 (채택)** — gmem 규약("현재 작업 브랜치로")과 사용자 지시를 그대로 따름.
- **근거**: 내 포크이고 문서 전용 커밋이라 되돌리기 쉽다. 브랜치를 임의로 바꾸면 사용자가 지시한
  범위를 말없이 변경하는 셈이 된다.
- **부작용**: upstream 재동기화(`git pull upstream release/v3.8.50` / rebase) 시 이 커밋이 얹혀
  있다. 충돌 자체는 `memory/` 가 upstream 에 없는 새 디렉토리라 거의 안 나지만, 히스토리는 갈라진다.
- **되돌리기**: `git revert <커밋>` 또는 `git reset --hard origin/release/v3.8.50` 후
  `git push --force-with-lease`.

---

## D-004 · `CLAUDE.md` 에 memory 포인터를 넣지 않는다

- **일자**: 2026-08-04 · **결정자**: Claude
- **맥락**: gmem 규약은 신규 합류자 발견성을 위해 `README.md` 또는 `CLAUDE.md` 에
  "작업 지식은 `memory/INDEX.md`" 한 줄을 넣으라고 한다.
- **결정**: **넣지 않았다.**
- **근거**: 이 저장소의 `CLAUDE.md` 는 47KB, `README.md` 는 102KB 짜리 **upstream 파일**이다.
  한 줄을 넣으면 upstream 동기화 때마다 충돌 지점이 하나 생긴다. 발견성 이득보다 유지 비용이 크다.
- **대신**: `memory/INDEX.md` 자체를 진입점으로 쓰고, 이 결정을 여기에 남긴다.
- **재검토 조건**: 이 포크를 독자적으로 유지·발전시키기로 하면(=upstream 동기화를 포기하면)
  그때 `CLAUDE.md` 상단에 한 줄 추가한다.

---

## D-005 · `package-lock.json` 변경분은 커밋하지 않는다

- **일자**: 2026-08-04 · **결정자**: Claude
- **맥락**: `npm install` 을 npm 11 로 돌린 결과 `package-lock.json` 이 1줄 바뀌었다.
  `"@types/bun": "*"` → `"@types/bun": "latest"` — npm 11 의 정규화다.
- **결정**: 커밋하지 않고 워킹 트리에 남겨 둔다.
- **근거**: 기능적 의미가 없는 도구 버전 차이(npm 9 ↔ 11)에서 온 잡음이고, 포크를 upstream 과
  불필요하게 갈라놓는다. gmem 규약도 문서 커밋과 코드 변경을 섞지 말라고 한다.
- **되돌리기**: `git checkout package-lock.json`.
  단, 이후 `npm ci` 를 npm 11 로 돌리면 다시 어긋날 수 있다.
