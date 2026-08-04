---
# the default layout is 'page'
title: AI Trade Manager
icon: fas fa-chart-line
order: 1
image: /assets/img/og/ai-trade-manager.jpg
---

<div align="center">
  <img src="assets/img/ai-trade-manager/dashboard-current.png" alt="AI Trade Manager 대시보드" width="1500">
</div>

> AI가 돈까지 벌어주면 좋겠지만 세상은 그렇게 만만하지 않다.

# 프로젝트를 시작한 이유
---

처음에는 RAG로 뉴스와 차트를 같이 읽어서 주가 전망을 해보는 정도를 생각했다. 뉴스와 현재 가격, 보유 종목을 같이 넘기면 AI가 그럴듯한 분석을 줄 수 있지 않을까 싶었다.

실제로 답은 나왔다. 문제는 그 답을 실제 투자 시스템에서 믿고 운영할 수 있느냐였다. 뉴스가 오래됐거나, 임베딩이 실패했거나, 현재 포트폴리오 상태를 잘못 읽어도 AI는 꽤 자연스러운 분석을 만들었다.

그래서 방향을 바꿨다. AI가 매수 버튼을 누르는 프로젝트가 아니라, AI 분석을 운영 가능한 자산 관리 흐름 안에 넣는 프로젝트로 만들기로 했다.

> 목표로 잡은 운영 기준

- 판단 근거와 사용된 데이터를 추적할 수 있어야 한다
- RAG와 provider 상태가 화면과 로그에 드러나야 한다
- paper/live 매매는 분리되고, 실전 BUY는 기본적으로 잠겨 있어야 한다
- 주문, 포지션, 포트폴리오 스냅샷은 나중에 다시 확인할 수 있어야 한다

> 전체 운영 기록은 [AI Trade Manager 허브](https://torpid-icon-d8a.notion.site/AI-Trade-Manager-3724054272b580d0b968f323059761da)와 [운영 설계 기록](https://torpid-icon-d8a.notion.site/AI-Trade-Manager-a704054272b583b3b1e081289e79cae2)에 있다.

<br/>

# 개발 타임라인
---

- 2026.01 — 거래 제어 MVP: Upbit 연동, Slack 명령, 매수/매도 확인
- 2026.02 — 서비스 구조화: FastAPI, PostgreSQL, SQLAlchemy, Alembic
- 2026.03 — 분석 자동화: APScheduler, AI 분석 API, 백테스트, OpenSearch RAG
- 2026.04 — AI 운영 콘솔: LangGraph AI 뱅커, Reviewer Agent, SSE 활동 추적
- 2026.05 — 운영 안정화: live BUY 잠금, paper/live 분리, provider fallback, RAG warning
- 2026.06 — 운영 검증 체계화: 트러블슈팅, AI 판단 계약, 화면 증거, RAG/DB/Test 근거 정리
- 2026.07 — 실주문 경계 재설계: 안전성 리뷰 P0 4건, 주문 멱등성, 전역 kill switch, fail-closed 거래 모드, UI 전면 개편

> 월별 상세는 [타임라인 DB](https://torpid-icon-d8a.notion.site/AI-Trade-Manager-d894054272b5827fb26d015d3ff14fee)에, 프로젝트 방향은 [목표와 운영 기준](https://torpid-icon-d8a.notion.site/3724054272b581caa9eeed5f55914b44)에 있다.

<br/>

# 전체 아키텍처
---

<div align="center">
  <img src="assets/img/ai-trade-manager/architecture.png" alt="AI Trade Manager 아키텍처" width="1900">
</div>

> 중요한 건 매수 버튼을 누르는 게 아니라, 누르면 안 될 때를 막는 것이었다.

| 영역 | 구성 | 역할 |
| :--- | :--- | :--- |
| Frontend | React/Vite, TanStack Query, Recharts | 대시보드, 포트폴리오, AI 뱅커, 백테스트 화면 |
| Backend | FastAPI, async SQLAlchemy, APScheduler | API, 스케줄러, 주문/분석 흐름 제어 |
| Data | PostgreSQL, OpenSearch | 포지션/주문/스냅샷 저장, 뉴스 RAG 검색 |
| Safety | 주문 의도 원장, Kill Switch Gate, 거래 모드 원장 | 실주문 멱등성, 신규 주문 차단, 모드 판정을 DB에서 결정 |
| External | Upbit, Slack, RSS/News | 거래소 연동, 운영 제어, 시장 컨텍스트 수집 |
| AI/Ops | Gemini, OpenAI fallback, Logs/Warning | 분석 생성, provider 장애 대응, 운영 상태 노출 |

Safety는 7월에 생긴 층이다. 주문을 낼지 말지를 애플리케이션 코드 여기저기서 판단하던 것을, 거래소로 나가는 모든 POST가 하나의 주문 서비스와 그 앞의 게이트를 지나도록 바꿨다. 프로세스가 재시작돼도 상태가 남아야 해서 메모리가 아니라 PostgreSQL에 뒀다.

> 기술 선택 이유는 [전체 아키텍처와 기술 선택](https://torpid-icon-d8a.notion.site/3724054272b581ee9cede0fe3899dc68)에 정리했다.

<br/>

# 운영과 안전장치
---

<div align="center">
  <img src="assets/img/ai-trade-manager/safety-flow.png" alt="AI Trade Manager 운영 안전장치 플로우" width="1900">
</div>

AI 자동매매라고 하면 보통 "AI가 알아서 사고판다"를 먼저 떠올리는데, 실제로는 "어떤 상황에서 절대 사지 못하게 할 것인가"가 더 중요했다.

판단 층에서는 reviewer와 entry gate가 근거 없는 매수를 거른다. BUY 직전 2차 검증은 비중을 줄일 수만 있고, 실행에는 저장된 분석 ID만 전달되며, provider가 늦으면 그 판단은 HOLD로 남는다.

주문 실행 층은 7월에 다시 만들었다.

| 위험 | 대응 |
| :--- | :--- |
| 타임아웃 재시도로 같은 주문이 두 번 체결 | 주문 의도를 DB에 먼저 저장하고 identifier 확정, 재전송 대신 조회로 복구 |
| 정지시켰는데 다음 스케줄에 주문이 나감 | 모든 거래소 POST 앞에 `ARMED` / `EXIT_ONLY` / `BLOCK_ALL` 게이트 |
| 설정 누락·오타가 live로 해석 | 기본값 `paper + inactive + BLOCK_ALL`, 판정 실패는 live로 보정하지 않음 |
| 전량청산 실패를 성공으로 표시 | 미체결 0, 잔고 0, 원장 일치를 확인한 `COMPLETED + VERIFIED`만 성공 |

실전 BUY는 기본적으로 비활성화했고, 잠금은 하나가 아니다. live 모드 전환, 봇 시작, 주문 게이트 `ARMED` 전환이 각각 별개의 관리자 작업이며 어느 것도 다음 단계를 자동으로 실행하지 않는다.

막지 말아야 할 것도 같이 정했다. 게이트가 닫혀 있어도 분석과 reconciliation은 돌고, 리스크 점검 실패로 막는 것은 신규 매수뿐이다. 매도와 비상청산까지 잠그면 자산을 빼지 못한다.

> 판단 계약과 주문 차단 조건은 [AI 판단 루틴과 프롬프트 계약](https://torpid-icon-d8a.notion.site/3754054272b581829337e86149b4b2e0), 장애 대응 근거는 [자동매매 안전장치와 장애 대응](https://torpid-icon-d8a.notion.site/3724054272b5818fac75e1fa743f654b)에 있다.

<br/>

# 문제와 트러블슈팅
---

운영하면서 잡은 케이스들이다. 초반 세 건은 노션에 정리해뒀다.

- AI 답변 신뢰성 — 데이터 품질 경고를 AI 답변보다 먼저 노출하게 했다
- 실전 주문 경계 — 검증 안 된 판단은 실제 주문 없이 paper/shadow 기록으로만 남긴다
- 비동기 작업 관측성 — 최종 응답만이 아니라 중간 상태와 실패 지점까지 화면과 로그에 남긴다

## 실주문 경계 재점검 (2026.07)

7월에는 판단 층 아래, 주문이 실제로 거래소에 나가는 구간을 처음부터 다시 읽었다. 결과가 좋지 않았다.

브로커 호출은 타임아웃이 나면 최대 5회까지 재시도하는데, 정작 주문에 고유 식별자를 붙이지 않고 있었다. 거래소가 주문을 접수하고 응답만 유실되면 같은 시장가 주문이 한 번 더 나갈 수 있는 구조였다. 비상 정지는 봇 설정의 플래그만 내렸고, 하드 TP/SL 경로는 그 플래그를 확인하기 전에 실행되고 있었다. 거래 모드 기본값은 `live`였고 정확히 `paper`가 아닌 값은 전부 live로 해석했다. 전량청산은 종목별 주문 예외를 로그로만 남기고 성공 응답을 돌려주고 있었다.

> 실제로 사고가 난 적은 없다. 다만 사고가 안 난 이유를 설계로 설명할 수 없었다.

네 건 모두 P0로 분류하고, 고치기 전까지 live 자동매매를 멈췄다. 그 결과물이 위 안전장치의 주문 실행 층이고, 고치는 동안 backend 테스트는 123개에서 632개가 됐다. 특히 걸린 건 세 번째였다. 운영 문서에는 모의 거래로 시작한다고 적어놨는데 코드 기본값은 live였다.

## 상태를 사실대로 보여주기 (2026.07)

같은 리뷰에서 프론트 문제도 나왔다. 포트폴리오 조회가 실패하면 화면에 0원이 찍혔다. 자산이 없는 것과 조회에 실패한 것이 구분되지 않았고, 그 0원은 AI 브리핑 컨텍스트로도 넘어갔다. AI는 "보유 자산이 없으니 신규 진입을 고려해볼 만하다" 같은 말을 자연스럽게 만들어낸다.

`live / snapshot / cached / error / empty / loading` 상태 판정을 순수 함수로 분리하고 전 화면이 같은 함수를 쓰게 했다. 캐시가 있는 오류는 값과 stale 경고를 같이 보여주고, 캐시가 없는 오류는 수치를 숨기고, 상태가 불확실하면 AI 전송을 막는다. 이 작업을 하면서 화면 전체를 공통 AppShell과 다크/라이트 semantic token으로 개편했다. `LIVE`나 `PARTIAL`은 색만으로 전달하지 않고 항상 텍스트를 붙였다.

> 케이스별 상세는 [운영 트러블슈팅](https://torpid-icon-d8a.notion.site/3734054272b5810d8903c9283ce1e6c1), [RAG와 AI Provider 운영](https://torpid-icon-d8a.notion.site/RAG-AI-Provider-3724054272b581aaa4d1db140cf36035), [DB Migration과 테스트 근거](https://torpid-icon-d8a.notion.site/DB-Migration-3724054272b581178dd3f280a7263f59)에 나눠 정리했다.

<br/>

# 주요 기능
---

## 대시보드

<div align="center">
  <img src="assets/img/ai-trade-manager/dashboard-current.png" alt="AI Trade Manager 메인 대시보드" width="1500">
</div>

> 시세보다 지금 시스템이 믿을 수 있는 상태인지 먼저 보여주는 화면이다.

시장 심리, 차트, AI 분석 요약, 포트폴리오 상태, provider warning을 한 화면에 모았다. 상단 바에는 거래 모드, runtime, 주문 게이트 상태가 어느 화면에서든 고정으로 떠 있다.

## AI 뱅커

<div align="center">
  <img src="assets/img/ai-trade-manager/ai-banker-chat.png" alt="AI 뱅커 채팅 화면" width="1500">
</div>

> AI의 최종 답변보다 답변이 만들어진 과정을 추적하게 만드는 것이 핵심이었다.

일반 챗봇처럼 바로 종목을 추천하지 않는다. LangGraph 기반으로 supervisor, RAG, quant, reviewer 역할을 나누고, 각 단계가 어떤 일을 했는지 SSE 활동 카드로 흘려보낸다. reviewer가 과도한 확신과 근거 부족을 한번 더 걸러낸다.

## 포트폴리오

<div align="center">
  <img src="assets/img/ai-trade-manager/portfolio-page.png" alt="포트폴리오 분석 화면" width="1500">
</div>

> 잔고 조회가 아니라 손익과 리스크를 시간 흐름으로 추적하는 화면이다.

Upbit 체결 내역과 잔고를 그대로 읽으면 예외가 많아서, 주문 이력과 포지션을 분리 저장하고 portfolio snapshot을 따로 남겼다. "언제부터 성과가 흔들렸는지"를 보려면 시점별 데이터가 필요하니까.

## 정책 연구소와 백테스트

<div align="center">
  <img src="assets/img/ai-trade-manager/laboratory-backtest.png" alt="정책 연구소와 백테스트 화면" width="1500">
</div>

> 그럴듯한 정책을 실전에 붙이기 전에 먼저 깨뜨려 보는 화면이다.

종목, 기간, 리스크 성향, 전략 preset으로 백테스트를 돌린다. 기대한 건 수익률 숫자가 아니라 정책이 어떤 조건에서 잘못되는지 찾는 것이다. 결과는 EMA/RSI 규칙 정책의 검증이지 운영 LLM 전략의 재현이 아니라서, 화면에도 그대로 명시했다.

## 설정

<div align="center">
  <img src="assets/img/ai-trade-manager/settings-page.png" alt="설정 화면" width="1500">
</div>

> 적용되지 않는 설정을 적용된 것처럼 보여주지 않는 화면이다.

PostgreSQL SystemConfig가 실제로 소비하는 키만 보여준다. 저장은 키별 서버 검증과 version 기반 CAS를 거치고, 충돌은 `409`로 거절된다. API key는 환경변수로만 읽고 화면에 노출하지 않는다.

> 화면별 검증 포인트는 [주요 기능과 화면](https://torpid-icon-d8a.notion.site/3724054272b58198880bee2d53c46587)에 정리했다.

<br/>

# 마무리
---

AI Trade Manager를 만들면서 느낀 건, AI 서비스에서 모델은 생각보다 일부라는 점이다. 실제로 운영하려면 데이터가 언제 들어왔는지, 실패하면 어디서 멈추는지, 실전 기능은 어떤 조건에서 잠겨야 하는지가 더 중요했다.

처음에는 RAG로 주가 전망을 해보고 싶어서 시작했다. 지금은 AI 분석, RAG 뉴스, 포트폴리오, 백테스트, 자동매매, Slack 제어를 하나의 운영 흐름으로 묶은 개인용 자산 관리 도구가 됐다. 7월 작업은 대부분 새 기능이 아니라, 이미 돌아가던 코드를 사고가 안 난 이유를 설명할 수 있는 상태로 바꾸는 일이었다. 재미있지는 않았지만 이번 달이 제일 남는 게 많았다.

아직 완성은 아니다. 원격 CI와 배포 경로 E2E, 파이썬 산술의 Decimal 전환, 단일 프로세스 스케줄러가 남아 있다. 다만 백엔드, AI, 데이터, 운영 설계를 하나의 흐름으로 연결하는 경험은 여기서 쌓았다.
