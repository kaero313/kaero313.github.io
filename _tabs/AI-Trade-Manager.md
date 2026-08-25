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

> AI가 돈까지 벌어주면 좋겠지만 돈은 못 벌더라. ~~대신 돈은 귀신같이 잘 먹음..~~

# 프로젝트를 시작한 이유
---

처음에는 RAG로 뉴스와 차트를 같이 읽어서 주가 전망을 해보는 정도를 생각했다. 뉴스와 현재 가격, 보유 종목을 같이 넘기면 AI가 그럴듯한 분석을 줄 수 있지 않을까 싶었다.

실제로 그럴싸한 답변은 해줬다. 문제는 그 답을 실제 투자 시스템에서 믿고 운영할 수 있느냐였다. 뉴스가 오래됐거나, 임베딩이 실패했거나, 
현재 포트폴리오 상태를 잘못 읽어도 AI는 꽤 자연스러운 분석을 만들었다. ~~그라믄 안돼~~ 

그래서 방향을 바꾸기로 했다. AI가 직접 매수 버튼을 누르는 **자동 매매 서비스**가 아니라, AI의 분석을 시스템의 **자산 관리 워크플로우 중 하나로 녹여내는 서비스**로 방향을 틀었다.

- 목표로 잡은 운영 기준
  - 판단 근거와 사용된 데이터를 추적할 수 있어야 한다
  - RAG와 AI 서비스 상태가 화면과 로그에 드러나야 한다
  - paper/live 매매는 분리되고, 실전 BUY는 기본적으로 잠겨 있어야 한다
  - 주문, 포지션, 포트폴리오 스냅샷은 나중에 다시 확인할 수 있어야 한다

- [개발 상세 기록 Notion](https://torpid-icon-d8a.notion.site/AI-Trade-Manager-3724054272b580d0b968f323059761da) &nbsp;&nbsp;/&nbsp;&nbsp; [운영 설계 기록](https://torpid-icon-d8a.notion.site/AI-Trade-Manager-a704054272b583b3b1e081289e79cae2)

<br/>

# 개발 타임라인
---

- 2026.01 — 거래 제어 MVP: Upbit 연동, Slack 명령, 매수/매도 확인
- 2026.02 — 서비스 구조화: FastAPI, PostgreSQL, SQLAlchemy, Alembic
- 2026.03 — 분석 자동화: AI 분석 API, 백테스트, OpenSearch RAG
- 2026.04 — AI 운영 콘솔: LangGraph AI 뱅커, 검토 에이전트, SSE 활동 추적
- 2026.05 — 운영 안정화: live BUY 잠금, paper/live 분리, AI 대체 전환, RAG 경고
- 2026.06 — 운영 검증 체계화: 트러블슈팅, AI 판단 계약, 화면 증거, RAG/DB/테스트 근거 정리
- 2026.07 — 실주문 경계 재설계: 안전성 리뷰, 주문, 전역 kill switch, fail-closed 거래 모드, UI 전면 개편

- [타임라인 DB](https://torpid-icon-d8a.notion.site/AI-Trade-Manager-d894054272b5827fb26d015d3ff14fee) &nbsp;&nbsp;/&nbsp;&nbsp; [프로젝트 목표와 운영 기준](https://torpid-icon-d8a.notion.site/3724054272b581caa9eeed5f55914b44)

<br/>

# 전체 아키텍처
---

<div align="center">
  <img src="assets/img/ai-trade-manager/architecture.png" alt="AI Trade Manager 아키텍처" width="1900">
</div>

> 중요한 건 매수 버튼을 누르는 게 아니라, 누르면 안 될 때를 막는 것이었다.

| 영역 | 구성 | 역할 |
| :--- | :--- | :--- |
| Frontend | React | 대시보드, 포트폴리오, AI 뱅커, 백테스트 화면 |
| Backend | FastAPI | API, 스케줄러, 주문/분석 흐름 제어 |
| Data | PostgreSQL, OpenSearch | 포지션/주문/스냅샷 저장, 뉴스 RAG 검색 |
| Safety | 주문 의도 / 거래 모드 관리, Kill Switch Gate | 신규 주문 차단, 모드 판정을 DB에서 결정 |
| External | Upbit, Slack, RSS/News | 거래소 연동, 운영 제어, 시장 컨텍스트 수집 |
| AI/Ops | AI 대체 전환, 로그/경고 | 분석 생성, 장애 대응, 운영 상태 노출 |

Safety 레이어는 이번 7월에 새로 구성했다. <br/>
주문을 낼지 말지를 앱 코드 여기저기서 판단하던 것을, 거래소로 나가는 모든 POST가 하나의 주문 서비스와 그 앞의 게이트를 지나도록 바꿨다. 그리고 프로세스가 재시작돼도 상태가 남아야 해서 메모리가 아니라 DB에 따로 두는게 좋다고 판단했다.

- [전체 아키텍처와 해당 기술을 선택한 이유](https://torpid-icon-d8a.notion.site/3724054272b581ee9cede0fe3899dc68)

<br/>

# 운영과 안전장치
---

<div align="center">
  <img src="assets/img/ai-trade-manager/safety-flow.png" alt="AI Trade Manager 운영 안전장치 플로우" width="1900">
</div>

AI 자동매매라고 하면 보통 "AI가 알아서 사고판다"를 먼저 떠올리는데, 실제로는 **어떤 상황에서 절대 사지 못하게 할 것인가**가 더 중요했다.

> 마치 하네스 엔지니어링 같은 느낌 ..?

판단 층에서는 검토 에이전트와 진입 게이트가 근거 없는 매수를 거른다. BUY 직전 2차 검증은 비중을 줄일 수만 있고, 실행에는 저장된 분석 ID만 전달되며, AI 서비스가 늦으면 그 판단은 HOLD로 남게 했다.

<br/>

| 위험 | 대응 |
| :--- | :--- |
| 타임아웃 재시도로 같은 주문이 두 번 체결 | 주문 의도를 DB에 먼저 저장하고 id 확정, 재전송 대신 조회로 복구 |
| 정지시켰는데 다음 스케줄에 주문이 나감 | 모든 거래소 POST 앞에 BLOCK 게이트 |
| 설정 누락·오타가 live로 해석 | 기본값 paper + BLOCK, 판정 실패는 live로 보정하지 않음 |
| 전량청산 실패를 성공으로 표시 | 미체결 0, 잔고 0, 원장 일치를 확인한 주문건만 성공 |

실전 BUY는 기본 상태값을 비활성화로 지정했다. live 모드 전환, 봇 시작, 주문 게이트 전환이 각각 별개의 관리자 작업이며 어느 것도 다음 단계를 자동으로 실행하지 않게 하여 안전성을 높히는 구조를 선택했다.

> 판단 계약과 주문 차단 조건은 [AI 판단 루틴과 프롬프트 계약](https://torpid-icon-d8a.notion.site/3754054272b581829337e86149b4b2e0),<br/> 장애 대응 근거는 [자동매매 안전장치와 장애 대응](https://torpid-icon-d8a.notion.site/3724054272b5818fac75e1fa743f654b)에 있다.

<br/>

# 문제와 트러블슈팅
---

걸린 것들이 대부분 같은 모양이었다. 기능이 안 되는 게 아니라, **잘 되는 것처럼 보이는데 근거가 없었다**.

봄에 겪은 것들이 그랬다. AI 답변은 뉴스가 오래됐든 검색이 실패했든 늘 자연스러웠고, 로그는 쌓이는데 정작 어디서 실패했는지는 못 찾았다.

제일 위험했던 건 포트폴리오였다. 조회가 실패하면 화면에 0원이 찍혔는데, 자산이 없는 것과 조회에 실패한 것이 구분되지 않았다. 그 0원은 화면에만 머무르지 않고 AI 브리핑으로 그대로 넘어갔다. AI는 그걸 받아서 "보유 자산이 없으니 신규 진입을 고려해볼 만하다"를 아주 자연스럽게 만들어냈다.

7월에는 주문이 실제로 거래소에 나가는 구간만 따로 떼서 처음부터 다시 읽었다. 상단의 위험·대응 표에 있는 네 건이 거기서 나왔고, 고칠 때까지 일단 실전 자동매매는 멈췄었다.

> 사고가 난 적은 없다. 다만 조건만 맞으면 언제든 날 수 있는 경로가 그대로 열려 있었다.

그중 제일 뜨끔했던 건 거래 모드였다. 운영 문서에는 모의 거래로 시작한다고 적어놨는데 코드 기본값은 실전이었고, 정확히 `paper`가 아닌 값은 전부 실전으로 해석하고 있었다. 오타 하나면 바로 실전이다. ~~드루와~~

> 케이스별 상세는 [운영 트러블슈팅](https://torpid-icon-d8a.notion.site/3734054272b5810d8903c9283ce1e6c1), [RAG와 AI 서비스 운영](https://torpid-icon-d8a.notion.site/3724054272b581aaa4d1db140cf36035), [DB Migration과 테스트 근거](https://torpid-icon-d8a.notion.site/3724054272b581178dd3f280a7263f59)에 나눠 정리했다.

<br/>

# 주요 기능
---

## 대시보드

<div align="center">
  <img src="assets/img/ai-trade-manager/dashboard-current.png" alt="AI Trade Manager 메인 대시보드" width="1500">
</div>

> AI 기반 시장 분석과 포트폴리오 리스크를 한눈에 관리하는 트레이딩 운영 화면

- 운영 상태 및 시장 심리와 트레이딩 차트
  - 공포탐욕 지수, 트렌드 강도, 변동성 경고를 묶어서 보여주고 RAG 뉴스로 탭 전환
- AI 판단 요약과 근거, 포트폴리오 요약
  - 결론만 주지 않고 기술 지표, 시장 심리처럼 항목별 근거를 같이 노출

## AI 뱅커

<div align="center">
  <img src="assets/img/ai-trade-manager/ai-banker-chat.png" alt="AI 뱅커 채팅 화면" width="1500">
</div>

> AI와 대화를 통하여 포트폴리오를 분석하는 화면

- 분석 파이프라인 멀티 에이전트
  - LangGraph로 supervisor, RAG, quant, reviewer를 나눠서 실행
- 근거를 갖춘 답변 형식
  - 시세 요약, 기술 지표, 시장 심리, 리스크 관측치 순으로 출력

## 포트폴리오

<div align="center">
  <img src="assets/img/ai-trade-manager/portfolio-page.png" alt="포트폴리오 분석 화면" width="1500">
</div>

> 잔고 조회 및 손익과 리스크를 시간 흐름으로 추적하는 화면

- 계좌 자산 현황 및 종목별 수익
  - 총 자산, 총 손익, 현금 잔고, 보유 종목 수를 Upbit 계좌 기준으로 출력
- 데이터 상태 표시
  - 실시간 여부와 마지막 조회 시각을 같이 띄워서, 조회 실패를 0원으로 보이지 않게 처리

## 정책 연구소와 백테스트

<div align="center">
  <img src="assets/img/ai-trade-manager/laboratory-backtest.png" alt="정책 연구소와 백테스트 화면" width="1500">
</div>

> 정책을 실전에 사용하기 전에 미리 테스트해보는 화면

- 실행 조건 지정, 정책 프리셋
  - 종목, 기간, 봉 단위, 초기 자본을 정해서 수행
- 결과 지표와 검증 리포트
  - 총 수익률, 최대 낙폭, 승률, 거래 수, 최종 자산을 같이 표출

## 설정

<div align="center">
  <img src="assets/img/ai-trade-manager/settings-page.png" alt="설정 화면" width="1500">
</div>

> 운영, 보안, API 등 전체 설정을 하는 화면

- 운영 설정 카테고리
  - 관리자 세션, AI 서비스, 매매 대상과 한도, BUY 안전락, 뉴스·비용, 리스크·스케줄 등
- AI 서비스
  - 모델, API, URL 등을 설정

> 화면별 검증 포인트는 [주요 기능과 화면](https://torpid-icon-d8a.notion.site/3724054272b58198880bee2d53c46587)에 정리했다.

<br/>

# 마무리
---

이 프로그램을 만들면서 느낀 건, AI 서비스에서 모델은 생각보다 일부라는 점이다. 실제로 운영하려면 **데이터**가 언제 들어왔는지, 실패하면 어디서 멈추는지, 실전 기능은 어떤 조건에서 잠겨야 하는지가 더 중요했다.

앞으로도 개발하고 운영하면서 생긴 이슈들이나 고민 사항들을 계속 다듬고 고도화 해 나갈 예정이다. 일단 다음 목표는 홈서버를 구축하고 거기에 올리고 운영해볼 예정인데, 벌써부터 난관이 예상되는구나.. ~~클라우드 가격 인하좀~~