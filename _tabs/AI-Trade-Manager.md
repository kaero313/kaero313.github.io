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
