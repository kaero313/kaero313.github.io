---
# the default layout is 'page'
title: AI Trade Manager
icon: fas fa-chart-line
order: 1
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

**상세 정리**

- 전체 운영 기록 허브: [AI Trade Manager](https://torpid-icon-d8a.notion.site/AI-Trade-Manager-3724054272b580d0b968f323059761da)
- 운영 설계 기록: [AI Trade Manager 운영 설계 기록](https://torpid-icon-d8a.notion.site/AI-Trade-Manager-a704054272b583b3b1e081289e79cae2)

<br/>

# 개발 타임라인
---

- **2026.01** 거래 제어 MVP: Upbit 연동, Slack/Telegram 명령, 매수/매도 확인
- **2026.02** 서비스 구조화: FastAPI, PostgreSQL, SQLAlchemy, Alembic
- **2026.03** 분석 자동화: APScheduler, AI 분석 API, 백테스트, OpenSearch RAG
- **2026.04** AI 운영 콘솔: LangGraph AI 뱅커, Reviewer Agent, SSE 활동 추적
- **2026.05** 운영 안정화: live BUY 잠금, paper/live 분리, provider fallback, RAG warning

**상세 정리**

- 월별 타임라인 DB: [AI Trade Manager 타임라인](https://torpid-icon-d8a.notion.site/AI-Trade-Manager-d894054272b5827fb26d015d3ff14fee)
- 프로젝트 목표와 운영 기준: [프로젝트 목표와 운영 기준](https://torpid-icon-d8a.notion.site/3724054272b581caa9eeed5f55914b44)

<br/>

# 최종 목표
---

AI Trade Manager의 목표는 종목을 추천하는 AI를 만드는 것이 아니었다.

AI 판단, 시장 데이터, 포트폴리오, 주문 실행, 운영 경고를 하나의 흐름으로 묶고, 실제 운영 중 문제가 생겼을 때 숨기지 않는 트레이딩 운영 콘솔을 만드는 것이 목표였다.

> 목표로 잡은 운영 기준

- 판단 근거와 사용된 데이터를 추적할 수 있어야 한다
- RAG와 provider 상태가 화면과 로그에 드러나야 한다
- paper/live 매매는 분리되고, 실전 BUY는 기본적으로 잠겨 있어야 한다
- 주문, 포지션, 포트폴리오 스냅샷은 나중에 다시 확인할 수 있어야 한다

결국 핵심은 AI가 무엇을 추천했는지가 아니라, 그 판단을 운영 환경에서 믿고 추적할 수 있는 구조를 만드는 것이었다. 이 기준을 만족시키기 위해 다음 아키텍처로 묶었다.

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
| External | Upbit, Slack, RSS/News | 거래소 연동, 운영 제어, 시장 컨텍스트 수집 |
| AI/Ops | Gemini, OpenAI fallback, Logs/Warning | 분석 생성, provider 장애 대응, 운영 상태 노출 |

AI 판단은 하나의 프롬프트로 끝내지 않고, 역할을 나눈 멀티 에이전트로 처리했다.

| 역할 | 책임 |
| :--- | :--- |
| Supervisor | 요청 분류와 작업 흐름 제어 |
| RAG Worker | 뉴스, 시장 문맥, RAG 검색 |
| Quant Worker | 가격, 지표, 포트폴리오 상태 분석 |
| Reviewer | 근거 부족, 과도한 확신, 주문 위험 검토 |

프론트는 단순 화면이 아니라 운영 콘솔 역할을 하도록 만들었다. AI 뱅커는 SSE로 에이전트 진행 상태를 보여주고, 대시보드는 시장 상태와 RAG/provider warning을 같이 보여준다.

백엔드는 FastAPI를 중심으로 주문, 분석, 스케줄러, 포트폴리오 스냅샷을 관리한다. PostgreSQL에는 추적 가능한 운영 데이터를 남기고, OpenSearch에는 뉴스 RAG 검색을 위한 chunk 문서와 parent 문서를 저장했다.

**상세 정리**

- 아키텍처 다이어그램과 기술 선택 이유: [전체 아키텍처와 기술 선택](https://torpid-icon-d8a.notion.site/3724054272b581ee9cede0fe3899dc68)

<br/>

# 운영과 안전장치
---

<div align="center">
  <img src="assets/img/ai-trade-manager/safety-flow.png" alt="AI Trade Manager 운영 안전장치 플로우" width="1900">
</div>

AI 자동매매라고 하면 보통 "AI가 알아서 사고판다"를 먼저 떠올리는데, 실제로는 "어떤 상황에서 절대 사지 못하게 할 것인가"가 더 중요했다.

| 위험 | 대응 |
| :--- | :--- |
| AI가 근거 부족한 매수를 제안 | reviewer agent와 confidence threshold로 차단 |
| 실전 매수가 의도치 않게 실행 | live BUY 기본 잠금, 명시적 설정 필요 |
| 정책 검증 없이 실전 반영 | paper trading과 policy backtest를 먼저 사용 |
| 과도한 비중 매수 | max buy weight, entry gate 적용 |
| provider 장애 | Gemini primary, OpenAI fallback, warning/log 저장 |
| RAG 데이터 부실 | stale 문서, embedding 실패, source health warning |

실전 BUY는 기본적으로 비활성화했다. 설정에서 명시적으로 풀기 전까지는 매수 주문이 나가지 않는다. shadow mode에서는 AI가 어떤 주문을 냈을지 기록만 하고 실제 체결은 하지 않는다.

이런 장치들은 화려한 기능은 아니지만, 운영형 AI 시스템에서는 오히려 이쪽이 더 중요하다고 생각한다. API를 붙이고 AI 답변을 받는 것보다, 실패했을 때 어디서 멈추고 무엇을 남길지 설계하는 일이 더 백엔드와 AI 서비스 운영에 가깝다.

**상세 정리**

- 운영 기준과 장애 대응 근거: [자동매매 안전장치와 장애 대응](https://torpid-icon-d8a.notion.site/3724054272b5818fac75e1fa743f654b)

<br/>

# 문제와 트러블슈팅
---

운영 기준을 잡고 나니 문제는 기능 구현보다 경계 관리에 가까웠다.

AI가 답을 만들고, 스케줄러가 돌고, 주문 API가 붙는 순간부터 중요한 건 "잘 동작한다"보다 "잘못 동작할 때 어디서 멈추고 무엇을 남길 것인가"였다.

## 1. AI 답변 신뢰성

| 항목 | 내용 |
| :--- | :--- |
| 운영 리스크 | 오래된 뉴스, 누락된 embedding, provider fallback 상황에서도 AI가 자연스러운 답변을 생성 |
| 조치 | ingestion 상태, source health, stale warning, missing embedding warning, provider warning 분리 |
| 남긴 기준 | AI 답변보다 데이터 품질과 provider 상태를 먼저 노출 |

## 2. 실전 주문 경계

| 항목 | 내용 |
| :--- | :--- |
| 운영 리스크 | 검증되지 않은 AI 판단이 실제 매수로 이어질 수 있음 |
| 조치 | paper/live 분리, live BUY lock, entry gate, TP/SL, max buy weight 적용 |
| 남긴 기준 | 실전 BUY는 기본 잠금, 검증되지 않은 판단은 paper/shadow에서만 기록 |

## 3. 비동기 작업 관측성

| 항목 | 내용 |
| :--- | :--- |
| 운영 리스크 | 스케줄러, RAG ingestion, AI provider 호출, reviewer 판단이 서로 다른 시점에 실행되어 실패 원인 추적이 어려움 |
| 조치 | analysis log, chat session, portfolio snapshot, provider warning, SSE activity UI 추가 |
| 남긴 기준 | 최종 응답뿐 아니라 중간 상태와 실패 지점까지 화면과 로그에 남김 |

**상세 정리**

- 운영 트러블슈팅 케이스 상세 정리: [운영 트러블슈팅 케이스](https://torpid-icon-d8a.notion.site/3734054272b5810d8903c9283ce1e6c1)
- RAG ingestion, embedding, provider fallback 기준: [RAG와 AI Provider 운영](https://torpid-icon-d8a.notion.site/RAG-AI-Provider-3724054272b581aaa4d1db140cf36035)
- DB migration과 테스트 검증 근거: [DB Migration과 테스트 근거](https://torpid-icon-d8a.notion.site/DB-Migration-3724054272b581178dd3f280a7263f59)

<br/>

# 운영 루틴과 AI 판단 계약
---

AI 판단은 주문 직전에 즉흥적으로 호출하는 구조가 아니다. 하루 동안 데이터 수집, 분석, 검증, 기록이 일정한 주기로 돌고, 주문 실행은 이미 저장된 AI 판단 로그와 안전장치를 다시 검사하는 단계로 분리했다.

- 운영 루틴

| 구분 | 운영 기준 |
| :--- | :--- |
| 뉴스/RAG 수집 | 설정값 기반 주기 실행, 코드 기본값은 4시간 |
| 시장 상태 | 5분마다 시장 심리지수 갱신 |
| AI 판단 | 60분마다 관심 종목별 `BUY / SELL / HOLD` 분석 |
| 검증/기록 | 30분마다 과거 판단 정확도 검증, 1시간마다 포트폴리오 스냅샷 저장 |
| 브리핑 | 매일 08:30 포트폴리오와 시장 상태 기반 AI 브리핑 |

AI 판단에는 포트폴리오, 기술 지표, RAG 뉴스, 시장 심리, 과거 실패 피드백을 넣었다. 출력은 주문 명령이 아니라 `decision`, `confidence`, `recommended_weight`, `reasoning`으로 구성된 판단 로그로 제한했다.

- 최종 프롬프트 요약

| 구분 | 내용 |
| :--- | :--- |
| System 영역 | 역할, JSON 스키마, 안전 규칙, 실패 피드백 |
| User 영역 | Symbol, Portfolio, Technical, News, Sentiment |
| Output | `BUY / SELL / HOLD`, `confidence`, `recommended_weight`, `reasoning` |
| Execution | AI 응답은 주문 요청이 아니라 분석 로그이며, executor가 confidence, entry gate, shadow mode, live BUY lock을 다시 검사 |

입력 컨텍스트는 보통 2,500~4,500 tokens 정도로 잡았다. 페르소나나 실패 피드백이 길어지면 5,000 tokens 이상으로 늘 수 있어서, 캔들 원문 전체가 아니라 계산된 지표와 요약된 근거를 넘기는 쪽으로 설계했다.

**상세 정리**

- AI 판단 루틴과 프롬프트 계약: [AI 판단 루틴과 프롬프트 계약](https://torpid-icon-d8a.notion.site/AI-3754054272b581829337e86149b4b2e0)

<br/>

# 주요 기능
---

이런 운영 기준을 화면에서는 다음 기능으로 풀었다.

## 대시보드

<div align="center">
  <img src="assets/img/ai-trade-manager/dashboard-current.png" alt="AI Trade Manager 메인 대시보드" width="1500">
</div>

> 시세보다 지금 시스템이 믿을 수 있는 상태인지 먼저 보여주는 화면이다.

대시보드는 현재 상태를 빠르게 보는 화면이다. 시장 심리, 관심 종목, 차트, AI 분석 요약, 포트폴리오 상태, provider warning을 한 화면에 모았다.

AI가 HOLD라고 했는지보다, 왜 HOLD인지와 현재 데이터가 정상인지가 더 중요하다고 봤다.

## AI 뱅커

<div align="center">
  <img src="assets/img/ai-trade-manager/ai-banker-chat.png" alt="AI 뱅커 채팅 화면" width="1500">
</div>

> AI의 최종 답변보다 답변이 만들어진 과정을 추적하게 만드는 것이 핵심이었다.

AI 뱅커는 일반 챗봇처럼 바로 종목을 추천하지 않는다. 현재 보유 자산, 시장 뉴스, 기술 지표, 리스크 상태를 확인하고 reviewer를 통해 과도한 확신이나 근거 부족을 한번 더 걸러낸다.

LangGraph 기반으로 supervisor, RAG, quantitative, reviewer 역할을 나누고, 각 단계가 어떤 일을 했는지 SSE 활동 카드로 흘려보내도록 했다.

## 포트폴리오

<div align="center">
  <img src="assets/img/ai-trade-manager/portfolio-page.png" alt="포트폴리오 분석 화면" width="1500">
</div>

> 잔고 조회가 아니라 손익과 리스크를 시간 흐름으로 추적하는 화면이다.

Upbit 체결 내역과 현재 잔고를 그대로 읽으면 생각보다 예외가 많았다. 그래서 주문 이력과 현재 포지션을 분리해서 저장하고, portfolio snapshot을 따로 남겼다.

화면에서는 총 자산, 보유 종목, 기간별 손익, AI 브리핑을 같이 보여준다. 나중에 "언제부터 성과가 흔들렸는지"를 보려면 스냅샷 데이터가 필요하다고 봤다.

## 정책 연구소와 백테스트

<div align="center">
  <img src="assets/img/ai-trade-manager/laboratory-backtest.png" alt="정책 연구소와 백테스트 화면" width="1500">
</div>

> 그럴듯한 정책을 실전에 붙이기 전에 먼저 깨뜨려 보는 화면이다.

정책 연구소에서는 종목, 기간, 리스크 성향, 전략 preset을 정하고 백테스트를 돌려볼 수 있다. 여기서 기대한 것은 수익률 숫자 하나가 아니라, 정책이 어떤 조건에서 잘못되는지를 찾는 것이었다.

**상세 정리**

- 화면별 기능과 검증 포인트: [주요 기능과 화면](https://torpid-icon-d8a.notion.site/3724054272b58198880bee2d53c46587)

<br/>

# 마무리
---

AI Trade Manager를 만들면서 느낀 건, AI 서비스에서 모델은 생각보다 일부라는 점이다.

모델을 잘 고르는 것도 중요하지만, 실제로 운영하려면 데이터가 언제 들어왔는지, 실패하면 어디서 멈추는지, 사용자가 어떤 근거를 보고 판단할 수 있는지, 실전 기능은 어떤 조건에서 잠겨야 하는지가 더 중요했다.

처음에는 RAG로 주가 전망을 해보고 싶어서 시작했다. 지금은 AI 분석, RAG 뉴스, 포트폴리오, 백테스트, 자동매매, Slack 제어를 하나의 운영 흐름으로 묶은 개인용 자산 관리 도구가 됐다.

아직 완성이라고 보기는 어렵다. 앞으로는 장기 성과 리포트, 더 정교한 리스크 엔진, 배포 환경의 관측성, 전략별 성과 비교를 더 보강할 생각이다. 다만 이 프로젝트를 통해 백엔드, AI, 데이터, 운영 설계를 한 화면에서 연결하는 경험은 꽤 많이 쌓았다고 생각한다.
