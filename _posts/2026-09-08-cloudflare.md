---
title: "[Cloudflare] Workers와 D1으로 블로그 방문 수 집계하기"
date: 2026-09-08 14:20:00 +0900
categories: [Infra, Cloud]
tags: [클라우드플레어, 인프라, 서버]
preview_image: /assets/img/cloudflare/cloudflare-logo.png
---

<div align="center">
  <img src="{{ '/assets/img/cloudflare/workers-d1-hero.png' | relative_url }}" alt="Cloudflare 로고와 Workers + D1 문구, D1 데이터베이스 일러스트를 조합한 이미지" width="1200">
</div>

현재 블로그는 GitHub Pages로 운영하고 있다. 여러 플랫폼을 봤지만 뭔가 깃허브랑 연관 되어있었으면 했고, 서버 관리도 필요 없어서 선택했었다. 
그대신 깃허브는 블로그 방문 수나 게시글 조회 수를 집계하는 기능이 없었다. 그래서 직접 만들기로 했다.

<br/>

# 구성
---

사용 중인 테마는 GoatCounter만 지원하고 있었다. 기존 카운터 위젯은 표시하려는 항목과 맞지 않아 집계 API를 따로 두기로 했다. 별도 서버를 운영하지 않고 API와 저장소를 연결할 수 있어 Workers와 D1을 선택했다.
<br/><br/>
브라우저가 페이지 로드 후 현재 경로를 Worker에 전송하면, Worker는 D1에 방문을 기록하고 집계 결과를 JSON으로 반환해주고 화면에서는 이 값을 사이드바와 게시글 상단에 표시한다.

<div align="center">
  <img src="{{ '/assets/img/cloudflare/architecture.png' | relative_url }}" alt="브라우저가 Worker에 경로를 보내면 D1에 방문을 기록하고 집계 결과를 반환하는 구조" width="1900">
</div>

<br/>

# 집계 방식
---

방문 기록은 페이지 경로와 시각을 저장하는 테이블 하나로 구성했다. 날짜와 경로 조건으로 방문 수를 집계해 별도의 카운터 초기화 없이 오늘·누적 방문 수를 구했다.

```sql
CREATE TABLE visits (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  path TEXT NOT NULL,
  visited_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

Worker는 방문 기록을 추가한 뒤 사이트 전체와 현재 페이지의 오늘·누적 건수를 반환한다. 배포에는 `wrangler deploy`를 사용했다.

<br/>

# 중복 집계와 로컬 테스트
---

## 30분 내 재방문 처리

페이지별 요청 시각과 응답을 `localStorage`에 저장해 30분 내 재방문은 API 요청 없이 기존 집계 값 표시

## 로컬에서는 예시 데이터 사용

CORS는 블로그 도메인으로 제한하고, 로컬 테스트는 API 호출 없이 예시 데이터를 사용해 실제 집계 제외

<br/>

# 적용 결과와 제약
---

사이드바에는 사이트의 오늘·누적 방문 수를, 게시글 상단에는 해당 글의 누적 조회 수를 표시한다.

이 수치는 고유 방문자 수가 아니라 중복 요청을 일부 제외한 방문 기록 수다. 시크릿 모드, 다른 기기, 브라우저 저장소 초기화는 별도 방문으로 집계되며, 서버에서 방문자를 식별하는 처리는 포함하지 않았다.

<br/>

# 후기
---

별도 서버를 관리하지 않고 필요한 집계 기능만 붙일 수 있어 개인 블로그에는 잘 맞았다. Workers에서 요청을 처리하고 D1에 기록을 저장하는 구성도 단순하게 유지할 수 있었다.
<br/><br/>
사실 이런 블로그에 방문자가 들어올 가능성은 거의 없기 때문에, 정확한 방문자 통계보다는 블로그와 글이 얼마나 읽히는지 가볍게 확인하는 용도로 사용하려고 한다.
