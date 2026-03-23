# PROJECT_ARCHITECTURE.md

## 1. 목적

이 문서는 `/home/ubuntu/.openclaw`와 그 하위 `workspace/`의 구조를 운영 관점에서 설명한다.

이 저장소는 단일 애플리케이션이 아니라 아래 요소가 결합된 개인 운영 허브다.

- OpenClaw 에이전트 런타임
- 자동화 크론 및 전달 큐
- 여러 정적 뉴스 보드
- Streamlit 기반 소형 데이터 도구
- 업비트 자동매매 템플릿과 보조 대시보드

핵심 특징은 "코드", "런타임 상태", "개인 설정", "운영 결과물"이 한 루트 아래 함께 존재한다는 점이다.

## 2. 상위 디렉토리 맥락

`/home/ubuntu` 아래에는 여러 개인 도구 디렉토리가 있고, `.openclaw`는 그중 하나다.

주요 인접 디렉토리:

- `/home/ubuntu/.codex`: Codex 세션, 규칙, 스킬 저장소
- `/home/ubuntu/.cache`: Python, Playwright, whisper 등 캐시
- `/home/ubuntu/.config`: 로컬 앱 설정
- `/home/ubuntu/.openclaw`: 이 문서의 분석 대상

즉 `.openclaw`는 독립 제품 저장소라기보다 개인 에이전트 운영 홈 디렉토리에 가깝다.

## 3. .openclaw 최상위 구조

### 설정/정체성

- `openclaw.json`: OpenClaw 런타임 메인 설정
- `openclaw.json.bak*`, `openclaw.json.old`, `openclaw.json.stable`: 설정 백업
- `exec-approvals.json`: 명령 실행 승인 상태

### 에이전트 런타임 데이터

- `agents/`: 에이전트 인증 정보, 세션 로그
- `cron/`: 예약 작업 정의와 실행 로그
- `delivery-queue/`: 외부 채널 전송 대기/실패 큐
- `logs/`: 런타임 감사 로그
- `memory/`: 런타임 메모리 DB 등 상태 파일

### 채널/장치/브라우저 상태

- `credentials/`: 채널 연동 관련 자격 정보
- `devices/`: 페어링 상태
- `identity/`: 디바이스 정체성
- `telegram/`: 텔레그램 관련 상태
- `browser/`: 브라우저 프로필 데이터

### 사용자 작업 공간

- `workspace/`: 실제 프로젝트와 문서, 이미지, 실험 결과물

### 기타

- `canvas/`: 간단한 정적 HTML 캔버스
- `completions/`: 셸 자동완성 스크립트
- `media/`: 인바운드 미디어 파일

## 4. 런타임 계층 설명

### 4.1 OpenClaw 설정 계층

`openclaw.json`은 다음 역할을 가진다.

- 기본 모델과 워크스페이스 루트 지정
- 오디오 전사 도구 설정
- 텔레그램 채널 활성화
- 로컬 게이트웨이 포트/토큰 설정
- 내부 훅과 플러그인 활성화

이 파일은 앱 설정 파일이면서 동시에 운영 비밀 저장소 역할도 하고 있다.

### 4.2 에이전트 세션 계층

`agents/main/agent/`에는 인증 정보가 있고, `agents/main/sessions/`에는 세션별 JSONL 로그가 쌓인다.

의미:

- 현재 대화 기록과 세션 이력 보존
- 에이전트 인증 상태 유지
- 삭제된 세션도 `*.deleted.*` 형태로 일부 잔존

### 4.3 스케줄러 계층

`cron/jobs.json`은 예약 작업 레지스트리다.

작업 유형:

- 뉴스 브리핑 생성
- 업비트 상태 점검
- 변동성 알림
- 자동매매 체결 알림

`cron/runs/`는 각 잡의 실행 결과 로그를 JSONL로 남긴다.

### 4.4 전달 계층

`delivery-queue/`는 에이전트 출력이 실제 채널 전송으로 이어지기 전후의 큐다.

구성:

- 루트: 전송 대기 또는 처리 중
- `failed/`: 실패한 전달 기록

### 4.5 사용자 워크스페이스 계층

`workspace/`는 Git 저장소이며, 실질적인 프로젝트 소스가 모여 있다.

다만 아래가 함께 섞여 있다.

- 문서
- 앱 코드
- 가상환경
- 데이터 캐시
- 이미지 산출물
- 휴지통

따라서 소스 저장소이면서 운영 결과물 저장소 역할도 동시에 한다.

## 5. workspace 구조

### 5.1 메타 문서

- `AGENTS.md`: 워크스페이스 운영 기본 규칙
- `SOUL.md`, `USER.md`, `IDENTITY.md`: 에이전트/사용자 정체성 문서
- `TOOLS.md`: 로컬 환경 메모
- `BOARD_OPS.md`: 뉴스 보드 운영 기준
- `HEARTBEAT.md`: heartbeat 동작 제어

이 문서들은 코드보다 운영 행동을 규정한다.

### 5.2 정적 뉴스 보드 계열

대상:

- `ai-news-homepage`
- `us-stock-homepage`
- `kr-stock-homepage`
- `kr-bonds-homepage`
- `world-news-homepage`
- `iran-war-homepage`
- `bixby-news-homepage`
- `main-homepage`

공통 패턴:

- `index.html`: 정적 골격
- `styles.css`: 스타일
- `app.js`: 클라이언트 렌더링/상호작용
- `news-data.js`: 콘텐츠 데이터 저장소
- `validate_news_sources.py`: 데이터 검증

운영 방식:

1. `workspace/*-homepage`에서 소스 수정
2. 검증 스크립트 실행
3. `/var/www/*-homepage`로 배포
4. nginx 라우팅을 통해 서비스

즉 이 디렉토리들은 빌드 시스템 없는 콘텐츠 게시판이다.

### 5.3 업비트 자동매매 계열

`upbit-trader/`는 가장 코드량이 많은 프로젝트다.

구성:

- `upbit_trader.py`: 수동 조회/주문 CLI
- `bot/`: 전략, 지표, 리스크, 상태 저장, 실행 엔진
- `scripts/`: paper/live 실행 스크립트와 그래프 생성
- `tests/`: 단위 테스트
- `web-dashboard/`: CSV 기반 Express 대시보드
- `logs/`, `state/`, `data/`: 실행 중 생성되는 상태/로그/시세 데이터

아키텍처 흐름:

1. `market_data.py`가 시세와 캔들 조회
2. `indicators.py`가 EMA/RSI/ADX 계산
3. `strategy.py`가 BUY/SELL/HOLD 판단
4. `risk.py`가 주문 가능 여부와 제한 계산
5. `engine.py`가 상태 로드, 판단, 실행, 로그 저장
6. 보조 크론이 `logs/`와 `state/`를 읽어 알림 생성

성격상 라이브 트레이딩 엔진과 운영 보고/알림 시스템이 결합되어 있다.

### 5.4 Streamlit 앱 계열

대상:

- `wto-summary-site`
- `wto-clone`
- `sp500-weekly`

특징:

- Python 단일 앱 구조
- 별도 백엔드 없이 Streamlit UI
- 일부는 `.venv`, `__pycache__`, 캐시 데이터 포함

세부:

- `wto-summary-site`: 엑셀 업로드 후 `Statement` 요약 컬럼 추가
- `wto-clone`: 유사 계열 Streamlit 실험본으로 보임
- `sp500-weekly`: 장기 S&P500 데이터 기반 시뮬레이터

운영 방식은 프로젝트별로 다르다.

- `wto-summary-site`: Dockerfile, docker-compose, systemd service 존재
- `sp500-weekly`: 로컬 실행 중심

### 5.5 실험/보조 프로젝트

- `sp500-pwa`: 정적 PWA 샘플
- `test-site`: 뉴스 갱신 실험용 소형 사이트
- 루트 PNG 파일들: 금융 차트 산출물

### 5.6 저장소 오염 요소

아래는 소스와 분리되지 않고 workspace에 직접 존재한다.

- `.venv-btc`, `.venv-chart`, `.venv-plot`
- `sp500-weekly/.venv`
- `wto-summary-site/.venv`
- `wto-clone/.venv`
- `__pycache__/`
- 대형 CSV/PNG 산출물
- `.trash/`

이 때문에 Git 저장소가 개발용 소스 트리와 데이터 저장소를 동시에 겸하고 있다.

## 6. 운영 데이터 흐름

### 6.1 뉴스 보드

뉴스 생성/수정
-> `news-data.js`
-> `validate_news_sources.py`
-> `/var/www/*`
-> nginx 라우팅
-> 브라우저 렌더링

### 6.2 업비트 자동매매

시장 데이터 조회
-> 전략/리스크 판단
-> 주문 실행 또는 스킵
-> `logs/trades.csv`, `logs/events.log`, `state/*.json`
-> 크론 잡이 로그 분석
-> 텔레그램 알림 전달

### 6.3 Streamlit 도구

브라우저 접속
-> Streamlit 앱
-> 업로드 파일 또는 로컬 캐시 처리
-> 다운로드 파일 또는 시각화 제공

### 6.4 OpenClaw 자동화

사용자 메시지 또는 cron wake
-> `agents` 세션 실행
-> 결과 생성
-> `delivery-queue`
-> 텔레그램 등 채널 전달

## 7. 배포/실행 모델

### 뉴스 보드

- 소스는 `workspace/`
- 서빙은 `/var/www/`
- 프런트는 정적 파일
- 배포는 수동 `rsync` 중심

### WTO Summary

두 가지 흔적이 공존한다.

- Docker 기반 실행 정의
- systemd 서비스 기반 실행 정의

즉 컨테이너화 시도와 호스트 직접 실행이 함께 존재한다.

### 업비트

- Python 스크립트 실행
- systemd/cron 기반 운영 가능성이 높음
- 로그 파일을 다른 자동화 잡이 다시 읽는 구조

## 8. Git 상태 해석

`workspace/`는 Git 저장소지만, 현재 아래 성격이 혼재한다.

- 추적 중인 문서/코드 수정
- 추적되지 않은 가상환경
- 산출 이미지
- 상태 JSON
- 캐시 데이터

이 저장소는 전형적인 "깨끗한 앱 리포지토리"가 아니라 운영 작업 디렉토리다.

## 9. 주요 리스크

### 9.1 비밀정보 평문 저장

확인된 범주:

- 텔레그램 봇 토큰
- 게이트웨이 토큰
- Notion API 키
- OAuth access/refresh 토큰

이들은 설정 파일과 인증 파일에 평문으로 존재한다.

### 9.2 코드와 상태의 혼합

문제:

- 런타임 결과물이 소스 트리에 바로 쌓임
- 어떤 파일이 배포 대상인지 구분이 어려움
- 백업과 Git 동기화 시 민감정보가 섞일 수 있음

### 9.3 배포 경계의 이중화

뉴스 보드는 `workspace`와 `/var/www`가 분리되어 있다.

영향:

- workspace만 수정하면 실제 반영 안 됨
- 코드 리뷰와 실제 서비스 상태가 어긋날 수 있음

### 9.4 포맷 의존 검증

`validate_news_sources.py`는 실제 JS 파서보다 문자열 패턴에 크게 의존한다.

영향:

- 포맷만 바뀌어도 검증 실패 가능
- 데이터 구조 변경에 취약

### 9.5 저장공간 관리

큰 용량을 차지하는 항목:

- `.trash/`
- 각종 `.venv`
- Streamlit/차트 데이터
- `.git`

운영이 길어질수록 관리 비용이 커질 가능성이 높다.

## 10. 실무적으로 보는 추천 정리 방향

### 우선순위 1

- 비밀정보를 파일 밖으로 이동
- `.env` 또는 시스템 시크릿 스토어 사용
- 인증/토큰 파일의 Git 포함 여부 점검

### 우선순위 2

- `workspace/`에서 가상환경과 산출물을 분리
- `runtime/`, `data/`, `artifacts/` 같은 별도 경로로 이동
- `.gitignore` 재정비

### 우선순위 3

- 뉴스 보드 배포를 스크립트 하나로 표준화
- 검증 + rsync + 소유권 변경 + 확인까지 묶기

### 우선순위 4

- OpenClaw 런타임 디렉토리와 제품 코드 디렉토리 역할 구분
- `.openclaw`는 운영 홈
- 실제 프로젝트는 별도 리포지토리로 분리 고려

## 11. 한 줄 요약

이 시스템은 "개인 AI 운영실"에 가깝다.

`.openclaw`는 에이전트 런타임과 자동화 허브이고, `workspace/`는 뉴스 게시판, 데이터 도구, 트레이딩 실험이 함께 들어 있는 작업 저장소다. 현재는 빠른 운영에는 유리하지만, 보안과 유지보수 관점에서는 경계 분리가 필요한 상태다.
