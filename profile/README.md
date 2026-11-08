# MOCOU

매주 월요일 10시, 월요병에 걸린 직장인에게 커피 쿠폰을 선착순으로 나눠 주는 서비스를 가정한 쿠폰 발급 시스템.

## 프로젝트 소개

MOCOU는 Monday Coupon의 약자다. 매주 월요일 정각에 커피 쿠폰 회차를 열고, 회차마다 10,000장을 1인 1매로 선착순 발급한다.

재고 10,000장에 20,000명이 정각에 동시에 요청하는 상황을 기준으로, 다음 세 가지를 지킨다.

- 재고보다 많이 발급하지 않는다.
- 한 회원에게 같은 회차 쿠폰을 두 장 주지 않는다.
- 발급됐다고 응답한 쿠폰은 DB에 반드시 남는다.

## 저장소

| 저장소 | 설명 |
|---|---|
| [backend](https://github.com/MO-COU/backend) | 선착순 발급, 비동기 저장과 장애 복구, 정합성 검증, 쿠폰 사용·만료, 관리자 API |
| [frontend](https://github.com/MO-COU/frontend) | 고객 발급 화면, 관리자 대시보드(발급 현황, 부하 테스트 실행, 정합성 검증) |
| [load-test](https://github.com/MO-COU/load-test) | 동시성 제어 6가지 방식 비교 실험 |

## 시스템 아키텍처

<img src="https://raw.githubusercontent.com/MO-COU/backend/main/docs/images/system-architecture.svg" width="100%" alt="MOCOU 시스템 아키텍처">

발급 요청은 Redis Lua 스크립트 하나가 원자적으로 판정하고 바로 응답한다. DB 저장은 Redis Stream 컨슈머가 비동기로 처리하고, 실패한 이벤트는 재처리와 DLQ로 복구한다. 인프라 구성과 발급 흐름은 [backend README](https://github.com/MO-COU/backend)에 있다.

## 핵심 기능

- **선착순 발급**: 20,000명이 재고 10,000장에 동시에 요청하는 조건에서 6가지 방식을 비교했다. DB에서 재고를 판정한 방식은 실패율 57-77%, Redis WATCH는 92%, Redis Lua는 0%였다. 20회 측정 모두 초과·중복 발급은 0건이다([load-test](https://github.com/MO-COU/load-test)).
- **비동기 저장과 유실 방지**: 운영 Redis는 AOF `appendfsync always`로 매 쓰기를 디스크에 남기고, 컨슈머는 DB 커밋 뒤에만 ACK한다.
- **정합성 검증**: 회원 100만·발급 300만 건을 한 스냅샷 위에서 규칙 8개로 검사한다. 검증 시간은 EC2에서 180초에서 90초로 줄였다.
- **부하 검증**: k6로 20,000 VU를 한꺼번에 보내는 조건을 6회 돌려 모두 오류 0건이었고, 워커 스레드 200에서 p95는 약 1.2초였다.

## 기술 스택

![Java](https://img.shields.io/badge/Java%2021-007396?style=flat&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot%204.1-6DB33F?style=flat&logo=springboot&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL%208.0-4479A1?style=flat&logo=mysql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis%208.8-DC382D?style=flat&logo=redis&logoColor=white)
![React](https://img.shields.io/badge/React%2019-20232A?style=flat&logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazonwebservices&logoColor=white)
![k6](https://img.shields.io/badge/k6-7D64FF?style=flat&logo=k6&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=flat&logo=githubactions&logoColor=white)

## 문서

- [아키텍처](https://github.com/MO-COU/backend/blob/main/docs/architecture.md) · [정합성 검증 규칙](https://github.com/MO-COU/backend/blob/main/docs/b1/consistency-rules.md)
- [쿠폰 생명주기 정책](https://github.com/MO-COU/backend/blob/main/docs/b2/coupon-lifecycle-policy.md) · [쿠폰 만료 배치](https://github.com/MO-COU/backend/blob/main/docs/b2/coupon-expiration-batch-guide.md)
- [동시성 제어 비교 결과](https://github.com/MO-COU/load-test/blob/main/docs/results.md)

## 팀원

| 이름 | 주요 담당 |
|---|---|
| 장근창 (팀장) | 대용량 데이터 생성, Redis-DB 정합성 검증과 검증 쿼리 튜닝, 동시성 제어 비교 실험 설계·측정 |
| 권도하 | 공통 기능 및 배포·운영 환경 |
| 권혁준 | 쿠폰 생명주기, 만료 Batch, Redis 발급 현황, 운영 로그 및 S3 백업 |
| 박서희 | 관리자 API, k6 부하 테스트 및 AWS SSM 기반 실행 |
| 조현빈 | Redis Stream Consumer, 재처리·DLQ, Transactional Outbox |
| 지창민 | Redis Lua 기반 발급, 재고·중복 제어, 발급 순번 및 Redis 상태 관리 |
