# OpenClaw History Reconstruction

이 문서는 `/home/ubuntu/ecode/openclaw` 저장소의 현재 구조와 Git 히스토리를 바탕으로, 프로젝트가 어떤 순서로 만들어지고 확장되었는지 재구성한 요약이다.

## 결론 요약

이 프로젝트는 처음부터 지금 같은 대형 멀티채널 플랫폼이 아니었다. 가장 초기 형태는 `warelay`라는 이름의 WhatsApp 릴레이 CLI였고, 이후 다음 흐름으로 확장된 것으로 보인다.

`WhatsApp 릴레이 CLI`
-> `자동응답/에이전트 실행기`
-> `Gateway 중심 아키텍처`
-> `macOS/iOS/Android 앱`
-> `Web UI`
-> `Skills`
-> `Plugin/Extension 기반 멀티채널 플랫폼`

또한 이름도 여러 번 바뀌었다.

- `warelay`
- `CLAWDIS` / `clawdis`
- `clawdbot`
- `moltbot`
- `openclaw` / `OpenClaw`

## 단계별 생성 순서

## 1. 시작점: 최소 저장소

- 루트 커밋은 `Initial commit`이며 내용은 `LICENSE`만 존재한다.
- 즉 저장소 자체는 빈 상태에서 시작했고, 제품 코드가 한 번에 대량 유입된 형태는 아니다.

대표 커밋:

- [`Initial commit`](https://github.com/beaur8-prog/openclaw/commit/f6dd362d39b8e30bd79ef7560aab9575712ccc11)

## 2. 2025-11-24: `warelay` CLI 탄생

초기 실질 커밋에서 다음 파일이 한 번에 생긴다.

- `README.md`
- `package.json`
- `src/index.ts`

커밋 메시지는 `Add warelay CLI with Twilio webhook support`였다. 이 시점의 프로젝트 정체성은 현재의 OpenClaw가 아니라, WhatsApp/Twilio 중심의 릴레이 도구였다.

핵심 특징:

- 단일 CLI 중심
- Twilio webhook 지원
- WhatsApp relay 목적
- 아직은 gateway, app, extension 구조 없음

대표 커밋:

- [`Add warelay CLI with Twilio webhook support`](https://github.com/beaur8-prog/openclaw/commit/16dfc1a5b929f7184261c52f50846daca536ca24)

## 3. 2025-11-24 ~ 2025-11-25: 빠른 모듈화

같은 날 안에 `src/`가 빠르게 분화된다.

등장한 축:

- `src/provider-web.ts`
- `src/utils.ts`
- `src/commands/*`
- `src/cli/*`
- `src/twilio/*`
- `src/auto-reply/*`
- `src/process/*`
- `src/media/*`
- 각종 `*.test.ts`

이 시기 확장 순서는 대체로 다음과 같다.

1. 단일 엔트리 파일
2. Web provider / Twilio provider 분리
3. CLI command 분리
4. auto-reply 로직 추가
5. process 실행과 queue 추가
6. media 처리 추가
7. test 보강

즉 지금의 코어는 처음부터 큰 설계로 나온 것이 아니라, 작동하는 CLI를 빠르게 분해하며 자란 형태다.

대표 커밋:

- [`Refactor CLI into modules for web provider and utils`](https://github.com/beaur8-prog/openclaw/commit/cafca5c42152c9edd006d2af9961aba1e19ed611)
- [`Add command modules and tests; commit remaining changes`](https://github.com/beaur8-prog/openclaw/commit/938e237411f8a5211a098c8dc6432f4c6901837b)
- [`feat: serialize command auto-replies with queue`](https://github.com/beaur8-prog/openclaw/commit/13be898c07e3df69e2118753b434ff9cecf7ab9c)
- [`feat: add image support across web and twilio`](https://github.com/beaur8-prog/openclaw/commit/948ff7f035f68810244c1698c9b090443cb5e7c2)

## 4. 초기 문서 생성: 기능 메모형 docs

`docs/`는 제품 소개 문서보다 기능 문서가 먼저 생긴다.

초기 생성 예:

- `docs/queue.md`
- `docs/images.md`
- `docs/audio.md`
- `docs/heartbeat.md`
- `docs/tmux.md`

이 시점의 문서는 현재의 정돈된 제품 문서 사이트라기보다, 방금 추가한 기능을 설명하는 실무 메모에 더 가깝다.

대표 커밋:

- [`feat: serialize command auto-replies with queue`](https://github.com/beaur8-prog/openclaw/commit/13be898c07e3df69e2118753b434ff9cecf7ab9c)
- [`feat: add image support across web and twilio`](https://github.com/beaur8-prog/openclaw/commit/948ff7f035f68810244c1698c9b090443cb5e7c2)
- [`feat: transcribe audio and surface transcript to prompts`](https://github.com/beaur8-prog/openclaw/commit/e642f128ae086fa1bc8ba5f513015114567a7f07)
- [`docs: document heartbeat triggers`](https://github.com/beaur8-prog/openclaw/commit/3998933b304e297afd7d92dc8840c6333a481288)
- [`CLI: rename heartbeat tmux helper and log file path`](https://github.com/beaur8-prog/openclaw/commit/7e5b3958ccbb4719c5f2db81fbbb259b7bf9e196)

## 5. 2025-11-26 ~ 2025-12-03: 릴레이에서 에이전트로 이동

이 구간부터 단순 relay를 넘어 agent 성격이 강화된다.

주요 변화:

- heartbeat 관련 기능
- typing indicator 유지
- thinking level 개념
- group message 처리
- direct agent run 관련 문서와 CLI

즉 프로젝트 목적이 단순 메시지 중계에서, “메시지 채널을 통해 동작하는 AI agent”로 이동하기 시작한다.

대표 커밋:

- [`feat: keep typing indicators alive during commands`](https://github.com/beaur8-prog/openclaw/commit/d871dad85f7a7f8fe389aa575c2739dc55fee47c)
- [`feat: add heartbeat cli and relay trigger`](https://github.com/beaur8-prog/openclaw/commit/271004bf60ad5354960fead300f99430e6b1f8cf)
- [`Agents: add pluggable CLIs`](https://github.com/beaur8-prog/openclaw/commit/f31e89d5af2ec84357a9e23ab748dc36c9fe6133)

## 6. 2025-12-03 ~ 2025-12-05: `CLAWDIS` 리브랜딩

`warelay`에서 `CLAWDIS`로 이름이 바뀌며 제품 문서가 본격적으로 생긴다.

이 시기 새로 생긴 대표 문서:

- `docs/index.md`
- `docs/configuration.md`
- `docs/agents.md`
- `docs/security.md`
- `docs/troubleshooting.md`
- `docs/lore.md`

이때부터는 단순 CLI 툴이 아니라, 설정과 운영, 보안, 구조를 갖춘 제품으로 성격이 바뀐 것으로 보인다.

대표 커밋:

- [`🦞 Rebrand to CLAWDIS - add docs, update README`](https://github.com/beaur8-prog/openclaw/commit/a27ee2366ebfa366ee960e567f72f97fe63085d9)
- [`Switch to clawdis RPC mode and complete rebrand`](https://github.com/beaur8-prog/openclaw/commit/b3e50cbb33a0fdc818603d3375c95f22b7a2ae05)
- [`chore: rename package to clawdis`](https://github.com/beaur8-prog/openclaw/commit/5949ef0e2ccc6a25888475473d3b9f6adc743cc2)

## 7. 2025-12-05 ~ 2025-12-10: Gateway 중심 구조 정착

이 시기의 변화는 현재 구조를 이해할 때 가장 중요하다.

핵심 이벤트:

- macOS companion app 추가
- `relay` 개념이 `gateway`로 전환
- WebSocket control plane 완성
- protocol / session / client 구조 정리

즉 현재 프로젝트의 뼈대는 초반 relay가 아니라, 이 시기에 자리 잡은 gateway 아키텍처다.

이 시점 이후의 많은 기능은 모두 gateway 위에 얹히는 방향으로 성장한다.

대표 커밋:

- [`feat: add mac companion app`](https://github.com/beaur8-prog/openclaw/commit/a5164df293bd6808e169c9eddc2282ac3b3804cf)
- [`chore: rename relay to gateway`](https://github.com/beaur8-prog/openclaw/commit/a3bf2bdd8cc3134ea84593e45d94f03d51e116fe)
- [`Gateway: finalize WS control plane`](https://github.com/beaur8-prog/openclaw/commit/b2e7fb01a926273e9d8fdac035d855e991c7465f)
- [`Gateway: discriminated protocol schema + CLI updates`](https://github.com/beaur8-prog/openclaw/commit/172ce6c79f400594501381f73d51105f6110db61)

## 8. 2025-12-12 ~ 2025-12-18: 모바일 앱 추가

모바일은 비교적 뒤늦게 들어온다.

### iOS

2025-12-12에 `apps/ios`가 scaffold로 등장한다.

초기 구성:

- bridge
- voice wake
- screen/canvas
- settings
- SwiftUI app skeleton

### Android

2025-12-14에 `apps/android`가 등장한다.

초기 구성:

- bridge
- canvas
- chat
- camera
- foreground service

의미:

- 앱이 먼저 있었던 것이 아니다.
- gateway/bridge 개념이 먼저 있고, 모바일은 그 뒤에 붙은 클라이언트다.

대표 커밋:

- [`feat(ios): add ClawdisNode app scaffold`](https://github.com/beaur8-prog/openclaw/commit/6d6c3ad2c45d28798183860701e9f03df8881795)
- [`feat(android): add Compose node app (bridge+canvas+chat+camera)`](https://github.com/beaur8-prog/openclaw/commit/b2378c01ea7d3615212c4670303be76634c86c8b)

## 9. 2025-12-18 ~ 2025-12-21: Web UI와 Skills 등장

### UI

`ui/`는 2025-12-18 `Gateway: add browser control UI`에서 처음 생긴다.

초기 UI는:

- `ui/index.html`
- `ui/src/main.ts`
- `ui/src/ui/*`
- `ui/src/styles.css`

곧이어 control dashboard가 확장되며 sessions, connections, cron, skills 등 여러 화면이 붙는다.

즉 지금의 웹 UI는 초창기 핵심이 아니라, gateway가 성숙한 뒤 올라간 관리 레이어다.

### Skills

`skills/`는 2025-12-20 `feat: add managed skills gating`에서 처음 대량 등장한다.

의미:

- skills는 코어 런타임의 출발점이 아니라
- 에이전트 사용성과 확장성을 높이기 위한 후속 계층이다

대표 커밋:

- [`Gateway: add browser control UI`](https://github.com/beaur8-prog/openclaw/commit/df0c51a63bf56cbdeb0035292eadbfcdfe3a95e7)
- [`feat: add managed skills gating`](https://github.com/beaur8-prog/openclaw/commit/d1850aaada68b50a188bfec57bf5d29eaa8bc492)
- [`feat(ui): expand control dashboard`](https://github.com/beaur8-prog/openclaw/commit/6ff837125475fe0f48e245dd721af947e870cb33)

## 10. 2026-01-04 ~ 2026-01-30: 이름이 다시 여러 번 변경

프로젝트 이름은 한 번만 바뀐 것이 아니다.

순서:

1. `warelay`
2. `clawdis`
3. `clawdbot`
4. `moltbot`
5. `openclaw`

특히 `refactor: rename to openclaw`는 2026-01-30에 등장한다. 따라서 현재의 OpenClaw 브랜딩은 프로젝트 역사상 꽤 후반부에 정착한 이름이다.

대표 커밋:

- [`🦞 Rebrand to CLAWDIS - add docs, update README`](https://github.com/beaur8-prog/openclaw/commit/a27ee2366ebfa366ee960e567f72f97fe63085d9)
- [`Rebrand app icons from Clawdis to Clawdbot`](https://github.com/beaur8-prog/openclaw/commit/fb0c566b2adc5914ba08340efced86540c73c617)
- [`refactor: rename to openclaw`](https://github.com/beaur8-prog/openclaw/commit/9a7160786a7dbd21469fad73992158e415e4686e)

## 11. 2026-01-11 이후: 플러그인 아키텍처 도입

`extensions/`는 매우 늦게 등장한다.

최초 이벤트:

- 2026-01-11 `feat: add plugin architecture`

처음 생긴 플러그인은 `extensions/voice-call`이다.

이후 순차적으로 추가되는 예:

- `extensions/zalo`
- `extensions/matrix`
- `extensions/msteams`

의미:

- 현재 저장소에서 비중이 큰 `extensions/*`는 초창기 구조가 아니다.
- 코어 gateway와 agent 구조가 먼저 형성된 뒤, 채널과 기능을 외부화하기 위해 plugin 구조가 도입되었다.

대표 커밋:

- [`feat: add plugin architecture`](https://github.com/beaur8-prog/openclaw/commit/cf0c72a557af74b96549b06ba3d13be82fcdf44a)
- [`feat: restore voice-call plugin parity`](https://github.com/beaur8-prog/openclaw/commit/42c17adb5e4d3ea1f9b1d2fd921b9abc183b79df)
- [`refactor: migrate extensions to plugin sdk`](https://github.com/beaur8-prog/openclaw/commit/1420d113d85c09956a7c38591b82be3e1489e089)

## 12. 2026-02 ~ 2026-03: 대형 멀티채널 플랫폼으로 폭발적 확장

이 구간에서는 다음이 크게 커진다.

- extension 수 증가
- plugin-sdk 표면 확대
- 문서 사이트 구조화
- 앱 기능 고도화
- CI, 릴리스, 운영 자동화
- 다양한 provider / channel / memory / tooling 통합

즉 현재 폴더의 “크기” 대부분은 이 후기 확장 단계에서 만들어진 것이다.

대표 커밋:

- [`UI: gateway dashboard with glassmorphism theme system`](https://github.com/beaur8-prog/openclaw/commit/3bbbe33a1b91c3cfe2327e2d5655c19c0b9fe3f8)
- [`Session management improvements and dashboard API`](https://github.com/beaur8-prog/openclaw/commit/7b61ca1b06154cfc630244bc9535da769608680a)
- [`refactor: privatize bundled sdk facades`](https://github.com/beaur8-prog/openclaw/commit/50ce9ac1c63dad1a4099b168a555fee406aaf00d)
- [`Voice Call: enforce spoken-output contract and fix stream TTS silence regression`](https://github.com/beaur8-prog/openclaw/commit/3f7f2c8dc96e353173eb5572b46775cff5b80147)

## 현재 디렉터리들을 역사 순서로 보면

대략적인 출현 순서는 다음과 같이 이해할 수 있다.

1. `src/`
- 가장 오래된 코어
- 처음부터 존재
- 단일 CLI에서 시작해 점차 모듈화

2. `docs/`
- 초기에는 기능 문서 위주
- 이후 제품 문서 사이트로 진화

3. `apps/macos`
- gateway 전환기 초반에 추가
- companion app 성격

4. `apps/ios`
- gateway/bridge 기반 위에 추가된 모바일 노드

5. `apps/android`
- iOS 직후 추가된 모바일 노드

6. `ui/`
- gateway 기반 control UI
- 코어보다 늦게 등장

7. `skills/`
- agent 사용성 확장을 위한 레이어

8. `extensions/`
- 가장 늦게 본격 도입된 구조
- 코어를 외부 plugin으로 분리/확장하는 단계

9. `packages/`
- 후반부 배포/호환성 목적의 별도 패키지 레이어

## 최종 해석

현재 OpenClaw를 “처음부터 거대한 AI 플랫폼”으로 보면 구조가 잘 이해되지 않는다. Git 이력을 보면 오히려 다음처럼 보는 편이 자연스럽다.

- 아주 작은 WhatsApp relay CLI에서 출발
- 자동응답과 agent 개념이 붙음
- gateway가 중심이 되는 control plane이 형성됨
- macOS, iOS, Android가 붙음
- Web UI와 skills가 생김
- 마지막으로 plugin architecture가 도입되며 대형 멀티채널 제품으로 커짐

즉 이 저장소의 핵심 성장 논리는:

`메시지 릴레이`
-> `에이전트`
-> `게이트웨이`
-> `클라이언트/노드`
-> `관리 UI`
-> `스킬`
-> `플러그인 기반 플랫폼`

으로 요약할 수 있다.
