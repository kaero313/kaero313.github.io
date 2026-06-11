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

