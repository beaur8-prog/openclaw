# 텔레그램 날씨 요청 처리 흐름

이 문서는 사용자가 텔레그램으로 `오늘 날씨 알려줘`라고 보냈을 때,
이 서버 안에서 어떤 순서로 처리되는지 설명합니다.

이번 버전은 다음을 함께 반영했습니다.

- 이 서버에 설치된 OpenClaw 코어 코드 확인 결과
- 실제 설정 파일과 세션 로그 확인 결과
- 실제 날씨 요청이 처리된 로그 확인 결과

가능하면 기술 용어는 풀어서 설명했고,
정확히 확인된 내용과 일부 추정이 섞이는 부분은 구분해서 적었습니다.

## 가장 짧은 요약

한 줄로 말하면 다음 순서입니다.

1. 사용자가 텔레그램으로 메시지를 보냅니다.
2. 이 서버에서 돌아가는 OpenClaw가 그 메시지를 받습니다.
3. 이 사용자가 봇과 대화할 권한이 있는지 확인합니다.
4. 이 사용자의 텔레그램 대화방을 하나의 세션으로 잡습니다.
5. 세션에 붙어 있는 스킬 목록을 보고 `weather` 스킬을 쓸 수 있는 상태로 만듭니다.
6. 모델이 이 요청을 날씨 질문으로 판단합니다.
7. 날씨 스킬 설명 파일을 읽습니다.
8. `wttr.in` 또는 `Open-Meteo` 같은 외부 날씨 서비스에서 데이터를 가져옵니다.
9. 답변 문장을 만든 뒤 텔레그램으로 다시 보냅니다.
10. 세션 기록과 텔레그램 업데이트 번호를 저장합니다.

## TL;DR

이 시스템은 크게 둘로 나뉩니다.

- `/home/ubuntu/.openclaw`
  - 설정, 세션 기록, 로그, 워크스페이스가 들어 있는 운영 디렉토리
- `/usr/lib/node_modules/openclaw`
  - 실제 OpenClaw 실행 코드가 설치된 위치

즉, 이 프로젝트 폴더만 보면 설정 저장소처럼 보이지만,
실제 텔레그램 처리 로직과 세션 처리 로직은 서버에 설치된 OpenClaw 코드가 담당합니다.

## OpenClaw 코어는 어디에 있나

이번에 확인한 결과 OpenClaw 코어는 이 서버에 로컬 설치되어 있습니다.

확인된 경로:

- 실행 파일: `/usr/bin/openclaw`
- 실제 연결 경로: `/usr/lib/node_modules/openclaw/openclaw.mjs`
- 패키지 루트: `/usr/lib/node_modules/openclaw`
- 컴파일된 실행 코드: `/usr/lib/node_modules/openclaw/dist`
- 텔레그램 확장 코드: `/usr/lib/node_modules/openclaw/extensions/telegram`
- 기본 제공 스킬: `/usr/lib/node_modules/openclaw/skills`

의미:

- 텔레그램 메시지 받기
- 보낸 사람 권한 확인
- 세션 만들기
- 스킬 목록 붙이기
- 에이전트 실행
- 텔레그램으로 답장 보내기

이런 핵심 흐름은 이 서버 안의 로컬 코드가 처리합니다.

반대로 외부로 나가는 것은 주로 아래입니다.

- 모델 호출: OpenAI 같은 모델 제공자
- 날씨 데이터 조회: `wttr.in`, `Open-Meteo`

## 확인된 주요 구성 요소

### 1. 텔레그램 채널이 켜져 있다

설정 파일 `openclaw.json`에서 텔레그램 채널이 켜져 있습니다.

중요한 값:

- `channels.telegram.enabled = true`
- `channels.telegram.dmPolicy = pairing`
- `channels.telegram.groupPolicy = open`
- `channels.telegram.streaming = off`
- `plugins.entries.telegram.enabled = true`

쉽게 말하면:

- 텔레그램 메시지를 받을 수 있음
- 개인 대화(DM)는 아무나 바로 쓰는 방식이 아니라, 허용된 사람만 쓰는 방식임
- 텔레그램 답변은 보통 중간 조각 없이 한 번에 가는 방식임

관련 파일:

- `openclaw.json`

### 2. 텔레그램 플러그인 자체가 로컬 코드로 있다

텔레그램 처리는 외부 서비스에 맡겨진 것이 아니라,
OpenClaw 안의 채널 플러그인으로 구현돼 있습니다.

관련 파일:

- `/usr/lib/node_modules/openclaw/extensions/telegram/index.ts`
- `/usr/lib/node_modules/openclaw/extensions/telegram/src/channel.ts`
- `/usr/lib/node_modules/openclaw/extensions/telegram/src/runtime.ts`

여기서 하는 일:

- `telegram` 채널 등록
- 텔레그램용 보안 정책 등록
- 페어링 승인 메시지 전송 방식 정의
- 그룹/스레드/메시지 액션 방식 정의

### 3. 누가 봇과 대화할 수 있는지 저장돼 있다

권한 관련 파일:

- `credentials/telegram-default-allowFrom.json`
- `credentials/telegram-pairing.json`

현재 확인된 내용:

- `allowFrom`에 `7587445335`가 들어 있음
- 페어링 요청 기록도 저장돼 있음

쉽게 말하면:

- 사용자가 처음 DM을 보냈을 때 바로 열리는 구조가 아니라
- 승인되거나 허용 목록에 들어간 사용자만 계속 대화할 수 있는 구조입니다.

### 4. 텔레그램에서 어디까지 읽었는지 저장한다

파일:

- `telegram/update-offset-default.json`

저장 내용:

- `lastUpdateId`
- `botId`

쉽게 말하면:

- 텔레그램 메시지를 읽을 때 "여기까지는 이미 처리했다"는 번호를 저장합니다.
- 그래서 같은 메시지를 중복으로 다시 처리하지 않게 합니다.

이건 추정이 아니라 로컬 코드로도 확인됩니다.

관련 코드:

- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:50770`
- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:51088`

### 5. 텔레그램 대화는 세션으로 유지된다

세션 파일:

- `agents/main/sessions/sessions.json`

실제 세션 키 예시:

- `agent:main:telegram:direct:7587445335`

이 세션 안에는 이런 값이 있습니다.

- `deliveryContext.channel = telegram`
- `deliveryContext.to = telegram:7587445335`
- `origin.provider = telegram`
- `origin.surface = telegram`
- `workspaceDir = /home/ubuntu/.openclaw/workspace`

쉽게 말하면:

- 같은 텔레그램 사용자와의 대화는 하나의 이어지는 대화로 관리됩니다.
- 사용자가 먼저 지역을 안 말하고,
  다음 메시지에서 `여긴 수원시 영통구야`라고 보내도
  같은 세션 안에서 이어서 처리할 수 있습니다.

세션 기록 관련 코드:

- `/usr/lib/node_modules/openclaw/dist/sessions-BEfs8c0Q.js:1922`

### 6. 세션에 스킬 목록이 붙는다

이 시스템은 세션마다 사용할 수 있는 스킬 목록을 붙여서 관리합니다.

확인된 스킬:

- `weather`

관련 파일:

- `workspace/AGENTS.md`
- `agents/main/sessions/sessions.json`
- `/usr/lib/node_modules/openclaw/skills/weather/SKILL.md`

관련 코드:

- `/usr/lib/node_modules/openclaw/dist/reply-CFQ8lILc.js:27708`
- `/usr/lib/node_modules/openclaw/dist/reply-CFQ8lILc.js:72391`
- `/usr/lib/node_modules/openclaw/dist/reply-CFQ8lILc.js:77471`

쉽게 말하면:

- 모델이 아무 것도 없는 상태에서 막 판단하는 것이 아니라
- "이런 일을 할 때는 이런 스킬을 읽어라"라는 안내를 같이 받습니다.
- 날씨 질문이면 `weather` 스킬을 읽을 수 있는 상태가 됩니다.

### 7. 실제 날씨 요청 로그도 있다

실제 세션 로그:

- `agents/main/sessions/d60b5a75-6ec5-4d23-afc0-b7f85b455221.jsonl`

확인된 실제 흐름:

1. 사용자가 음성 메시지를 보냄
2. 음성 전사 결과가 텍스트로 들어옴
3. 에이전트가 의도를 다시 확인함
4. 사용자가 지역을 보냄: `여긴 수원시 영통구야`
5. 에이전트가 `weather` 스킬 파일을 읽음
6. `wttr.in` 호출 시도
7. Open-Meteo도 함께 호출
8. 같은 텔레그램 세션으로 답변 보냄

즉, 날씨 질문은 실제로 이미 이 구조를 타고 동작한 적이 있습니다.

## 실제 처리 순서

### 1단계. 사용자가 텔레그램으로 메시지를 보낸다

예:

- `오늘 날씨 알려줘`

그럼 텔레그램 봇으로 업데이트가 들어옵니다.

관련 코드:

- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:51088`
- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:48689`

설명:

- OpenClaw는 텔레그램을 폴링 방식으로 받을 수도 있고
- 웹훅 방식으로 받을 수도 있습니다.
- 현재 상태 파일과 코드상으로는 두 방식 모두 지원합니다.

### 2단계. 이 사용자가 대화 가능한지 확인한다

검사 대상:

- `dmPolicy`
- `allowFrom`
- pairing 저장소

관련 코드:

- `/usr/lib/node_modules/openclaw/extensions/telegram/src/channel.ts`
- `/usr/lib/node_modules/openclaw/dist/send-6vB8BriV.js`
- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:47820`

설명:

- 허용되지 않은 사용자의 DM이면 여기서 막힐 수 있습니다.
- 그룹 메시지라면 그룹 허용 정책도 같이 봅니다.
- 버튼 클릭, 반응(이모지), 콜백도 따로 권한 체크가 있습니다.

### 3단계. 어느 세션으로 넣을지 결정한다

관련 코드:

- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:48763`
- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:48783`
- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:49165`

설명:

- 텔레그램 메시지로부터 `ctxPayload`라는 내부 입력 정보를 만듭니다.
- 여기에는 보낸 사람, 채널, 스레드, 본문, 미디어, 위치 정보 등이 들어갑니다.
- 그 다음 `resolveAgentRoute(...)`로
  어느 에이전트와 어느 세션으로 보낼지 정합니다.
- 그리고 `recordInboundSession(...)`으로 세션 정보를 저장합니다.

쉽게 말하면:

- "이 메시지는 누구 것이고, 어느 대화방 것이고, 이전 대화와 이어지는가?"를 정리하는 단계입니다.

### 4단계. 날씨 질문으로 판단할 준비를 한다

관련 파일:

- `/usr/lib/node_modules/openclaw/skills/weather/SKILL.md`

설명:

- 세션에는 이미 사용할 수 있는 스킬 목록이 붙어 있습니다.
- 날씨 관련 질문이면 모델은 `weather` 스킬을 읽는 쪽으로 유도됩니다.
- 이 스킬 파일에는 어떤 사이트를 쓰고 어떤 식으로 질의할지 안내가 적혀 있습니다.

예를 들어 이 스킬은 주로 다음을 안내합니다.

- `wttr.in`
- `Open-Meteo`
- `curl`

### 5단계. 실제 에이전트 실행으로 넘긴다

관련 코드:

- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:49578`
- `/usr/lib/node_modules/openclaw/dist/reply-CFQ8lILc.js:72391`
- `/usr/lib/node_modules/openclaw/dist/reply-CFQ8lILc.js:77471`

설명:

- Telegram 전용 메시지 처리기가 준비한 입력을
  공통 에이전트 실행 파이프라인으로 넘깁니다.
- 이 파이프라인이 실제 모델 호출, 도구 호출, 스킬 사용을 조정합니다.

쉽게 말하면:

- 텔레그램 전용 앞단에서 메시지를 정리한 뒤
- 공통 AI 엔진에게 "이거 처리해"라고 넘기는 단계입니다.

### 6단계. 날씨 데이터를 가져온다

실제 로그에서 확인된 방식:

- `curl -s "wttr.in/Suwon+Yeongtong?format=j1"`
- Open-Meteo API 호출

설명:

- 간단한 날씨는 `wttr.in`으로 처리할 수 있고
- 조금 더 구조화된 예보 정보는 Open-Meteo를 쓸 수 있습니다.
- 어떤 쪽을 쓸지는 스킬 설명과 모델 판단에 따라 달라질 수 있습니다.

중요한 구분:

- "어떤 외부 날씨 서비스를 고를지"는 모델 판단 요소가 있음
- 하지만 "텔레그램에서 받아서 세션에 넣고, 도구를 실행하고, 다시 답장 보내는 것"은 로컬 OpenClaw 코드가 담당함

### 7단계. 답변을 만들고 텔레그램으로 다시 보낸다

관련 코드:

- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:49560`
- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:49980`
- `/usr/lib/node_modules/openclaw/dist/send-6vB8BriV.js`

설명:

- Telegram 전용 답변 디스패처가 동작합니다.
- 필요하면 typing 표시, 반응 이모지, 메시지 수정, 미리보기 메시지 등을 처리할 수 있습니다.
- 최종적으로는 텔레그램 채팅방 ID와 스레드 정보를 써서 답장을 보냅니다.

현재 설정상:

- `channels.telegram.streaming = off`

즉, 사용자 입장에서는 보통 중간 스트리밍 텍스트보다 최종 답변을 한 번에 받는 형태에 가깝습니다.

### 8단계. 처리 결과를 저장한다

저장 위치:

- `agents/main/sessions/*.jsonl`
- `agents/main/sessions/sessions.json`
- `telegram/update-offset-default.json`

설명:

- 어떤 세션에서 무슨 대화를 했는지 저장합니다.
- 텔레그램에서 어디까지 읽었는지도 저장합니다.
- 다음 메시지가 오면 이어서 처리할 수 있게 됩니다.

## 실제 코드 경로 요약

`오늘 날씨 알려줘` 같은 일반 Telegram 메시지는 대략 아래 코드 경로를 탑니다.

1. 텔레그램 플러그인 등록
   - `/usr/lib/node_modules/openclaw/extensions/telegram/index.ts`
2. 텔레그램 polling/webhook 시작
   - `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:50770`
   - `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:51088`
3. Telegram `message` 이벤트 수신
   - `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:48689`
4. 메시지 문맥 만들기
   - `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:48763`
5. 세션 및 라우팅 결정
   - `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:48783`
6. 세션 메타 저장
   - `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:49165`
7. 텔레그램 메시지 dispatch
   - `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js:49578`
8. 공통 agent 실행
   - `/usr/lib/node_modules/openclaw/dist/reply-CFQ8lILc.js:72391`
   - `/usr/lib/node_modules/openclaw/dist/reply-CFQ8lILc.js:77471`
9. Telegram 전송 유틸로 답장
   - `/usr/lib/node_modules/openclaw/dist/send-6vB8BriV.js`

## 날씨 요청, 뉴스 요청, 일반 대화의 차이

### A. 날씨 요청

예:

- `오늘 날씨 알려줘`
- `내일 수원 날씨 알려줘`

공통 흐름:

1. Telegram 수신
2. 권한 확인
3. 세션 결정
4. `weather` 스킬을 쓸 수 있는 상태에서 agent 실행
5. 외부 날씨 서비스 호출
6. 답변 생성
7. Telegram 전송

특징:

- 스킬이 비교적 명확함
- 외부 데이터가 거의 반드시 필요함
- 위치 정보가 부족하면 추가 질문이 나올 수 있음

### B. 뉴스 요청

예:

- `오늘 AI 뉴스 요약해줘`
- `미국장 뉴스 알려줘`

공통 흐름:

1. Telegram 수신
2. 권한 확인
3. 세션 결정
4. agent 실행
5. 뉴스 데이터를 웹 검색, 저장된 자료, 또는 로컬 워크스페이스 결과물에서 가져옴
6. 답변 생성
7. Telegram 전송

차이점:

- 날씨처럼 전용 `weather` 스킬이 바로 걸리는 구조와 달리
- 뉴스는 웹 검색, 기존 배치 결과, 워크스페이스 파일 활용 등 경로가 더 다양할 수 있음
- 특히 이 프로젝트는 `workspace/*-homepage`와 크론 작업이 많아서
  이미 생성된 뉴스 산출물을 참조할 가능성도 큽니다.

즉:

- 날씨는 "실시간 조회형"
- 뉴스는 "실시간 조회 + 기존 배치 결과 활용형"

### C. 일반 대화

예:

- `안녕`
- `이거 어떻게 생각해?`
- `오늘 일정 정리해줘`

공통 흐름:

1. Telegram 수신
2. 권한 확인
3. 세션 결정
4. agent 실행
5. 필요하면 스킬 또는 도구 사용
6. 답변 생성
7. Telegram 전송

차이점:

- 날씨처럼 전용 데이터 조회가 꼭 필요하지 않을 수 있음
- 단순 대화면 외부 API 호출 없이 바로 답할 수도 있음
- 다만 명령 성격이 강하거나 특정 작업이면 다른 스킬을 쓸 수 있음

## 세 가지를 한 표로 보면

| 유형 | 예시 | 외부 데이터 필요성 | 전용 스킬 가능성 | 이 프로젝트 특성 |
| --- | --- | --- | --- | --- |
| 날씨 요청 | 오늘 날씨 알려줘 | 높음 | 높음 (`weather`) | 실시간 조회 비중 큼 |
| 뉴스 요청 | 오늘 AI 뉴스 요약해줘 | 높음 | 중간 | 웹 조회 + 기존 배치 산출물 활용 가능 |
| 일반 대화 | 안녕, 뭐하고 있어 | 낮음 | 낮음 | 세션 문맥만으로 답할 수 있음 |

## 이 문서에서 확인된 것과 아직 추정인 것

### 확인된 것

- OpenClaw 코어는 이 서버에 로컬 설치돼 있다.
- Telegram은 로컬 플러그인 코드로 처리된다.
- DM 허용 정책과 allowFrom 저장소가 실제로 동작한다.
- Telegram update offset 저장 코드가 있다.
- Telegram 메시지 수신 핸들러가 로컬 코드에 있다.
- 세션 메타를 저장하는 로컬 코드가 있다.
- 세션에 스킬 스냅샷을 붙이는 코드가 있다.
- 실제 텔레그램 날씨 요청 로그에서 `weather` 스킬과 외부 날씨 조회가 확인됐다.

### 아직 일부 추정인 것

- 한 턴에서 `wttr.in`과 `Open-Meteo` 중 무엇을 먼저 쓸지의 세부 판단
- 번들된 dist 파일 내부에서 세부 함수 경계가 어떻게 나뉘는지 100% 사람 읽기 좋게 복원한 수준
- 모든 뉴스 요청이 항상 같은 데이터 경로를 타는지 여부

## 관련 파일

운영 상태와 설정:

- `openclaw.json`
- `credentials/telegram-default-allowFrom.json`
- `credentials/telegram-pairing.json`
- `telegram/update-offset-default.json`
- `agents/main/sessions/sessions.json`
- `agents/main/sessions/d60b5a75-6ec5-4d23-afc0-b7f85b455221.jsonl`
- `workspace/AGENTS.md`

로컬 설치된 OpenClaw 코어:

- `/usr/bin/openclaw`
- `/usr/lib/node_modules/openclaw/openclaw.mjs`
- `/usr/lib/node_modules/openclaw/package.json`
- `/usr/lib/node_modules/openclaw/extensions/telegram/index.ts`
- `/usr/lib/node_modules/openclaw/extensions/telegram/src/channel.ts`
- `/usr/lib/node_modules/openclaw/extensions/telegram/src/runtime.ts`
- `/usr/lib/node_modules/openclaw/dist/send-6vB8BriV.js`
- `/usr/lib/node_modules/openclaw/dist/sessions-BEfs8c0Q.js`
- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/reply-D-26Je1S.js`
- `/usr/lib/node_modules/openclaw/dist/reply-CFQ8lILc.js`
- `/usr/lib/node_modules/openclaw/skills/weather/SKILL.md`

## 실무적으로 이해하면

이 시스템은 아래처럼 보면 됩니다.

- Telegram은 입력 채널
- OpenClaw 로컬 코드는 관제실
- 세션 저장소는 대화 기억 장치
- 스킬은 "이런 종류의 질문은 이렇게 처리해라"라는 사용 설명서
- 외부 API는 실제 데이터 공급원

그래서 `오늘 날씨 알려줘`는 사실상 이렇게 이해하면 됩니다.

`텔레그램 메시지 -> 로컬 OpenClaw -> 권한 확인 -> 세션 확인 -> weather 스킬 사용 -> 외부 날씨 조회 -> 답변 생성 -> 텔레그램 회신`

