
# 🎟️ MOCOU

> 대규모 동시 요청 환경에서 초과 발급을 방지하고, 1인 1매와 Redis-DB 간 데이터 정합성을 보장하는 선착순 쿠폰 발급 시스템

MOCOU는 단순한 쿠폰 CRUD 구현을 넘어 **동시성 제어 · 비동기 처리 · 장애 복구 · 데이터 정합성 · 부하 테스트 · 운영**
까지 검증한 프로젝트입니다.

---

## 📦 Repositories

| Repository | 설명 |
|---|---|
| [backend](https://github.com/MO-COU/backend) | Redis Lua 기반 선착순 발급, Redis Stream 비동기 처리, 정합성 검증, 쿠폰 생명주기, 관리자 및 운영 기능 |
| [frontend](https://github.com/MO-COU/frontend) | 관리자 대시보드 및 운영·테스트 결과 조회 UI |
| [load-test](https://github.com/MO-COU/load-test) | 6가지 동시성 제어 전략 비교 및 Redis Lua 선정 실험 |

---

## 🎯 Project Goals

선착순 쿠폰 발급 환경에서 다음 문제를 해결하는 것을 목표로 했습니다.

- 쿠폰 재고 이상의 초과 발급 방지
- 동일 회원의 중복 발급 방지
- 높은 동시 요청 환경에서 DB 경합 최소화
- Redis 예약과 MySQL 최종 데이터 간 정합성 보장
- 비동기 DB 저장 과정에서 발생하는 장애 복구
- 실제 부하 테스트를 통한 기술 선택 및 시스템 검증
- 운영 환경에서의 로그 보존과 장애 분석

---

## 🏗 System Architecture

<!-- 시스템 아키텍처 이미지 삽입 -->
<!-- 예시:
<img src="이미지_URL" width="100%" alt="MOCOU System Architecture">
-->
<img src="https://github.com/user-attachments/assets/2992ba1d-bad5-4e39-ab93-ede97f1bddca" width="100%" alt="MOCOU 시스템 아키텍처">

MOCOU는 Redis Lua를 이용해 쿠폰 발급 가능 여부, 중복 발급, 재고 차감을 원자적으로 처리하고, Redis Stream을 통해 MySQL 저장을 비동기로 수행합니다.

Consumer 처리 실패 시 PEL과 재시도, DLQ를 통해 장애 이벤트를 관리하며, 최종적으로 Redis와 MySQL 데이터를 교차 검증하여 정합성을 확인합니다.

👉 [Backend Repository](https://github.com/MO-COU/backend)

👉 [상세 아키텍처 문서](https://github.com/MO-COU/backend/blob/main/docs/architecture.md)

---

## 🔬 Architecture Decision

최종 발급 방식을 처음부터 Redis Lua로 결정하지 않고, 동일한 조건에서 6가지 동시성 제어 전략을 직접 구현하고 비교했습니다.

| 전략 | DB | Redis |
|---|---|---|
| 대기 | Pessimistic Lock | Redisson RLock |
| 재시도 | Optimistic Lock | Redis WATCH |
| 원자적 처리 | Atomic UPDATE | Redis Lua |

실험 결과 단순히 Redis를 사용하는 것보다 **재고 판정 자체를 DB에서 Redis로 이동하는 것이 병목 개선의 핵심**임을 확인했습니다.

이에 따라 최종 시스템의 동시성 제어 방식으로 **Redis Lua 기반 원자적 발급 구조**를 채택했습니다.

👉 [동시성 제어 6가지 비교 실험](https://github.com/MO-COU/load-test)

---

## 🔥 Core Features

### Redis Lua 기반 원자적 발급

Redis Lua Script에서 다음 작업을 하나의 원자적 연산으로 처리합니다.

- 쿠폰 발급 가능 시간 검증
- 동일 회원 중복 발급 검증
- 잔여 재고 확인 및 차감
- 발급 회원 등록
- 예약 순번 생성
- Redis Stream 이벤트 생성

이를 통해 별도의 분산 락 없이 동시 요청 간 재고 정합성을 유지합니다.

---

### Redis Stream 기반 비동기 DB 저장

Redis에서 발급 예약을 완료한 뒤, MySQL 저장은 Redis Stream Consumer가 비동기로 처리합니다.

사용자 요청 경로에서 DB 저장을 분리해 DB 처리 시간이 API 응답시간에 직접 영향을 주는 것을 줄였습니다.

---

### PEL · Retry · DLQ 기반 장애 복구

Consumer가 이벤트를 읽은 뒤 DB 처리에 실패한 경우 ACK되지 않은 이벤트는 Pending Entries List에 남습니다.

이후 재처리를 수행하고, 지속적으로 실패하는 이벤트는 DLQ로 분리하여 정상 이벤트 처리에 영향을 주지 않도록 구성했습니다.

---

### Redis ↔ MySQL 정합성 검증

비동기 구조에서는 Redis 예약 상태와 MySQL 최종 상태 사이에 차이가 발생할 수 있기 때문에 별도의 정합성 검증 기능을 구현했습니다.

주요 검증 항목은 다음과 같습니다.

- 초과 발급 여부
- 동일 회원 중복 발급 여부
- DB 발급 수와 잔여 재고 관계
- Redis 예약 회원과 DB 발급 회원 비교
- 쿠폰 상태와 상태 변경 이력 검증
- Redis 예약 순번 및 발급 당시 잔여 재고 검증

---

### Transactional Outbox 기반 알림 처리

쿠폰 발급이나 사용 처리와 알림 데이터를 동일한 DB Transaction 안에서 저장하고, 실제 알림 처리는 별도 Dispatcher에서 수행합니다.

이를 통해 DB Commit 이후 서버 장애로 인해 알림 정보가 유실되는 문제를 줄였습니다.

---

### Coupon Lifecycle

쿠폰은 발급 이후 다음 상태로 전이됩니다.

```text
ISSUED
  ├──→ USED
  └──→ EXPIRED
```

쿠폰 사용은 조건부 상태 변경을 통해 중복 사용을 방지하고, 만료 처리는 Spring Batch를 이용해 대량으로 처리합니다.

---

## 📊 Load Test

MOCOU에서는 부하 테스트를 두 가지 목적으로 나누어 수행했습니다.

### 1. 동시성 제어 전략 비교

최종 시스템 개발 전, 동일한 조건에서 6가지 동시성 제어 방식을 비교했습니다.

이 실험을 통해 Redis Lua 기반 구조를 최종 발급 방식으로 선정했습니다.

회원 20,000명이 재고 10,000장에 동시에 몰리는 조건에서 Redis Lua만 실패율 0%로 12초 안에 재고를 모두 발급했습니다. 재고를 DB에서 판정한 방식은 60초 타임아웃에 걸리며 실패율이 57-77%였습니다.

👉 [MO-COU/load-test](https://github.com/MO-COU/load-test)

---

### 2. 최종 시스템 부하 검증

최종 시스템에서는 k6를 이용해 여러 형태의 트래픽을 재현했습니다.

* 20,000명 Ramp-up
* 20,000·50,000 VU Spike
* Ramp Once
* Constant RPS
* 동일 회원 반복 요청

기본 쿠폰 재고는 **10,000장**을 기준으로 검증했습니다.

### 주요 검증 기준

```text
쿠폰별 회원 발급 수 ≤ 1

발급 수 ≤ 총 재고

DB 발급 수 + DB 잔여 재고 = 총 재고

Redis 발급 회원 = DB 발급 회원, Redis 재고 = DB 잔여 재고
```

이 밖에 상태·시각 모순, 상태 이력 체인, 참조 무결성, 예약 순번을 검사합니다. Redis Stream 미처리 이벤트·Consumer Pending·DLQ가 남아 있으면 판정하지 않고 `ERROR`로 기록합니다.

상세 시나리오와 테스트 구현은 Backend 및 Load Test 저장소에서 확인할 수 있습니다.

👉 [Backend Repository](https://github.com/MO-COU/backend)

👉 [Load Test Repository](https://github.com/MO-COU/load-test)

---

## 🛠 Tech Stack

### Backend

![Java](https://img.shields.io/badge/Java-007396?style=flat\&logo=openjdk\&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-6DB33F?style=flat\&logo=springboot\&logoColor=white)
![Spring Batch](https://img.shields.io/badge/Spring%20Batch-6DB33F?style=flat\&logo=spring\&logoColor=white)
![Gradle](https://img.shields.io/badge/Gradle-02303A?style=flat\&logo=gradle\&logoColor=white)

### Database & Messaging

![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat\&logo=mysql\&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat\&logo=redis\&logoColor=white)
![Flyway](https://img.shields.io/badge/Flyway-CC0200?style=flat\&logo=flyway\&logoColor=white)

### Infrastructure

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat\&logo=docker\&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS%20EC2-FF9900?style=flat\&logo=amazonec2\&logoColor=white)
![AWS S3](https://img.shields.io/badge/AWS%20S3-569A31?style=flat\&logo=amazons3\&logoColor=white)
![AWS SSM](https://img.shields.io/badge/AWS%20SSM-232F3E?style=flat\&logo=amazonwebservices\&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=flat\&logo=githubactions\&logoColor=white)

### Test

![JUnit5](https://img.shields.io/badge/JUnit%205-25A162?style=flat\&logo=junit5\&logoColor=white)
![Testcontainers](https://img.shields.io/badge/Testcontainers-2496ED?style=flat\&logo=testcontainers\&logoColor=white)
![k6](https://img.shields.io/badge/k6-7D64FF?style=flat\&logo=k6\&logoColor=white)

---

## 👥 Team

| 이름  | 주요 담당                                                |
| --- | ---------------------------------------------------- |
| 권도하 | 공통 기능 및 배포·운영 환경                                     |
| 권혁준 | 쿠폰 생명주기, 만료 Batch, Redis 발급 현황, 운영 로그 및 S3 백업        |
| 박서희 | 관리자 API, k6 부하 테스트 및 AWS SSM 기반 실행                   |
| 장근창 (팀장) | 대용량 데이터 생성, Redis-DB 정합성 검증과 검증 쿼리 튜닝, 동시성 제어 비교 실험 설계·측정 |
| 조현빈 | Redis Stream Consumer, 재처리·DLQ, Transactional Outbox |
| 지창민 | Redis Lua 기반 발급, 재고·중복 제어, 발급 순번 및 Redis 상태 관리       |

---

## 📚 Documents

### Backend

* [Backend README](https://github.com/MO-COU/backend)
* [System Architecture](https://github.com/MO-COU/backend/blob/main/docs/architecture.md)
* [Consistency Rules](https://github.com/MO-COU/backend/blob/main/docs/b1/consistency-rules.md)
* [Coupon Lifecycle](https://github.com/MO-COU/backend/blob/main/docs/b2/coupon-lifecycle-policy.md)
* [Expiration Batch](https://github.com/MO-COU/backend/blob/main/docs/b2/coupon-expiration-batch-guide.md)

### Load Test

* [Concurrency Strategy Comparison](https://github.com/MO-COU/load-test)
* [Detailed Test Results](https://github.com/MO-COU/load-test/blob/main/docs/results.md)

### Frontend

* [Frontend Repository](https://github.com/MO-COU/frontend)

---

## MOCOU

**선착순이라는 하나의 문제를 동시성 제어에서 끝내지 않고 비동기 처리, 데이터 정합성, 장애 복구, 부하 테스트와 운영까지 확장하여 검증한 프로젝트입니다.**

