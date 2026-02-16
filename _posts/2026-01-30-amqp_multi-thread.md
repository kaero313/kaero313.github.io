---
title: "[AMQP] Multi-Thread를 통한 대용량 데이터 처리"
date: 2026-01-30 15:21:44 +0900
categories: [Infra, MQ]
tags: [AMQP, 인프라, 메세지큐, MQ]
preview_image: assets/img/amqp/prev_icon.png
---

<div align="center">
  <img src="assets/img/amqp/amqp_logo.png" alt="AMQP 아이콘" width="500">
</div>
> RabbitMQ 아이콘을 쓸까 하다가 그냥 이걸로 했다.

이전에 스마트홈 관련 서비스를 운영했던 적이 있었다. 아파트 가구에 스마트 기기를 등록해서 가구별로 직접 앱을 이용하여 상태를 제어할 수 있는 서비스였다.<br/>
앱을 처음에 로그인 할 때 로그인 한 계정의 세대에 존재하는 모든 기기를 불러오는 로직이 있었는데, 이 부분에서 수십개의 제어 장치를 앱에서 부터 시작하여
서비스용 백앤드 서버 > 게이트웨이 서버 > 홈넷 서버 > 실물 장치로 health check를 한 뒤, 역순으로 다시 올라오는 과정이 있었기 때문에 속도가 상당히 느려 
고객의 불만이 자주 나오고, 다른 요청들과 맞물리면 가끔 타임아웃이 발생하여 실물 장치 로드에 실패하는 경우도 있었다.<br/>
요청이 소실되는 문제를 방지하고, 순간적인 과부화를 줄이기 위해 MQ를 도입하기로 결정했었다.<br/>
<br/>

# MQ를 이용한 대량의 데이터 처리
---

## 도입 목적

- 비동기, 확장성, 보증성
- 부하 분산 처리
- 데이터 손실 방지

<br/>

## 기존 처리 방식

<div align="center">
  <img src="assets/img/amqp/before_workflow.png" alt="도입 이전 워크플로우" width="500">
</div>
> 요청이 들어오면 중간 과정 없이 그대로 서버가 받아서 처리하는 구조

- 처리해야 할 데이터가 많아지니 과부하 발생
- 그에 따라 정상적으로 처리되지않고 유실되는 데이터 발생

<br/>

## 개선 방안

<div align="center">
  <img src="assets/img/amqp/after_workflow.png" alt="도입 이후 워크플로우" width="500">
</div>
> AMQP 도입, 중간에서 분산 처리하는 구조로 개선

- 처리해야 할 데이터를 분산해줌으로써, 프로세스 안정화
- 결론적으로, 전송된 메세지의 보장성 제공

## 테스트 수행

- Apache JMeter을 사용하여 부하 테스트 진행
  - 초당 5000건, 60초간 총 300,000건의 Request

- AMQP 도입 전

<div align="center">
  <img src="assets/img/amqp/before_test.png" alt="도입 이전 테스트" width="500">
</div>
> 프로세스 과부하로 인하여 요청 전체를 처리하지 못 하고 Timeout이 발생한 데이터가 생김

- AMQP 도입 후

<div align="center">
  <img src="assets/img/amqp/after_test.png" alt="도입 이후 테스트" width="500">
</div>
> 리스닝중인 프로세스가 과부하가 발생하지 않게 관리하여 유실된 데이터 없이 전체 처리 완료

# 멀티 스레드와 스레드 풀
---

## 도입 목적

MQ로 부하 분산 처리를 하였지만 허용량 이상의 요청이 들어오거나 로직 자체가 복잡하여 수행 시간이 오래 걸린다면 결론적으로 과부하가 발생하는 문제 개선