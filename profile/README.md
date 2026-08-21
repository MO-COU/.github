# MO-COU

대규모 트래픽 환경에서 **초과 발급 0건 · 1인 1매 · 정합성**을 보장하는 선착순 쿠폰 발급 시스템.

## 저장소 구성

| 저장소 | 역할 |
|---|---|
| [`backend`](https://github.com/MO-COU/backend) | 실제 서비스 코드. `main`은 배포용, `dev`는 개발용 |
| [`frontend`](https://github.com/MO-COU/fronted) | 관리자 대시보드를 보여주는 프론트앤드 |
| [`load-test`](https://github.com/MO-COU/load-test) | 동시성 제어 방식 선정용 실험 저장소 |


## 팀 구성

| 팀 | 담당 영역 |
|---|---|
| A | 발급·트래픽 제어 (Redis 재고, 1인 1매, 대기열, 발급 실패 처리) |
| B | 쿠폰 생명주기·정합성 (상태 전이, 만료 배치, 정합성 검증) |
| C | 관리자·관측·테스트 (대시보드, Mock 알림, 부하 테스트, CI/CD) |

---

## 아키텍처 선정 — `load-test`의 실측 근거

선착순 발급의 핵심은 "동시 요청이 몰릴 때 정합성을 어떻게 지키는가"다. 이걸 실 서비스 배포 파이프라인 위에서 이것저것 실험하면 위험하고 느리다. 그래서 별도 저장소에서 테스트를 진행했다.

- 같은 요구사항을 **6가지 동시성 제어 방식**(대기·재시도·원자적 처리 × DB·Redis)으로 각각 구현하고
- 동일한 부하 조건(재고 10,000 / 회원 20,000명 / VU 20,000, 60초 ramp-up)에서 정합성 → 성능 순으로 실측 비교한 뒤
- 그 결과를 근거로 `backend`에 반영할 방식을 결정했다.

| 전략 | DB | Redis |
|---|---|---|
| **대기** — 락을 걸고 시작 | `pessimistic-lock` | `redisson-rlock` |
| **재시도** — 충돌하면 다시 | `optimistic` | `redis-watch` |
| **원자적** — 한 번에 처리 | `atomic-update` | `redis-lua` |

정합성(초과 발급 0건, 중복 발급 0건)을 통과한 방식만 성능 비교 대상에 올렸고, 처리량(`issued`/60초)·커넥션 풀 고갈·워커 스레드 포화 지표를 비교했다.

**→ `redis-lua` 채택 완료.** 상세 비교 방법과 결과는 [`load-test`](https://github.com/MO-COU/load-test) 저장소 참고.

### 왜 원자적 Redis 처리인가

- 대기(락) 계열은 커넥션 풀과 워커 스레드가 락 대기로 고갈되기 쉽다.
- 재시도 계열은 충돌이 잦은 선착순 시나리오에서 재시도가 눈덩이처럼 불어난다.
- Redis 원자적 처리(Lua)는 재고 확인·차감·1인 1매 체크를 하나의 스크립트로 묶어, DB 락 경합 없이 순서를 보장한다.

## `backend`의 핵심 설계 원칙

`load-test`에서 검증한 건 "재고를 어떻게 원자적으로 깎는가"이고, 실제 서비스에는 그 결정이 안전하게 뚫리지 않도록 하는 방어선이 층층이 있다.

- **1차 방어 (Redis)**: Lua 스크립트로 재고 확인·차감·1인 1매 체크를 원자적으로 처리
- **최종 방어 (DB)**: `coupon_issue`에 `UNIQUE(coupon_id, member_id)` — Redis가 어떤 이유로든 뚫려도 DB가 물리적으로 중복 발급을 차단
- **멱등성**: 상태 변경(사용/취소) API는 클라이언트가 생성한 멱등키를 저장해 재시도에 안전하게 대응
- **정합성 검증**: 재고·발급 이력·Redis 상태를 사후 대조하는 배치를 별도로 두고, 실행 결과를 DB에 이력으로 남김 (`verification_run` → `verification_rule_result` → `verification_violation`)

---

## 기술 스택

| 구분 | 선택 |
|---|---|
| 언어 | Java 21 |
| 프레임워크 | Spring Boot 4.1.0-RC1 |
| 빌드 | Gradle (Kotlin DSL) 9.0 |
| DB | MySQL 8.0 |
| 캐시/동시성 | Redis 8.8 (Lettuce) |
| 배치 | Spring Batch 6.0.3 |
| 마이그레이션 | Flyway |
| 인프라 | Docker Compose, AWS EC2 (SSM 기반 배포) |
| 테스트 | JUnit 5, Testcontainers, AssertJ |
| 부하테스트 | k6 |
| CI/CD | GitHub Actions |

## `backend` 프로젝트 구조

```
src/main/java/com/mocou/
├── global/          # 공통 (응답 포맷, 예외 처리, 요청 추적, 마스킹)
├── member/          # 회원 도메인 (공통)
├── coupon/          # 쿠폰/재고 마스터 도메인 (공통)
├── issue/           # 발급·트래픽 제어 (A팀)
├── lifecycle/       # 쿠폰 생명주기 (B팀)
├── consistency/     # 정합성 검증 (B팀)
├── admin/           # 관리자 대시보드 (C팀)
├── notification/    # Mock 알림 (C팀)
└── datagen/         # 더미데이터 생성 배치 (B팀)
```

패키지 소유권 규칙: `global`/`member`/`coupon`은 읽기 전용 공통 패키지이고, 다른 팀 패키지는 직접 import하지 않는다 — 데이터가 필요하면 공통 패키지를 참조하거나 상대 팀이 공개한 인터페이스(`NotificationSender` 등)를 통해서만 접근한다.

## DB 스키마

회원/쿠폰/발급/이력/알림/실패로그/검증 3종 — 총 10개 테이블. 상세는 [`backend/src/main/resources/db/migration`](https://github.com/MO-COU/backend/tree/main/src/main/resources/db/migration) 참고.

## 시작하기

각 저장소의 README 참고:
- 서비스 실행: [`backend/README.md`](https://github.com/MO-COU/backend/blob/main/README.md)
- 동시성 실험 재현: [`load-test/README.md`](https://github.com/MO-COU/load-test/blob/main/README.md)
---
<details>
<summary>멘토링 질문 (8월 18일)</summary>

<p><b>Q1.</b><br>
저희가 이해한 선착순 쿠폰 시나리오는 오픈 시각에 사용자가 한꺼번에 몰리는 상황입니다. 그런데 60초에 걸쳐 VU를 0에서 20,000까지 점진적으로 늘리면, 초반에는 동시 접속이 적어서 경합이 약하게 발생합니다. 재고가 램프업 도중에 소진되면 뒷구간은 전부 재고 소진 응답이 되고요. 부하 테스트 조건으로 "테스트 유저 20,000명, ramp-up 60초"를 받았는데, 램프업의 의도가 뭔지 궁금합니다.</p>

<p><b>Q2.</b><br>
저희 프로젝트에서는 테스트 결과를 근거로 기술 선택의 타당성을 보여주려고 합니다.
예를 들어 Redis 도입 전후를 비교해서 동시성 처리나 성능 측면에서 Redis가 필요한지 검증할 수 있다고 생각합니다. 그런데 이런 식으로 접근하면 DB 종류(MySQL, PostgreSQL 등)나 서버 프레임워크(Spring, Node.js 등)까지 모두 비교 테스트를 해야 하는지 의문이 듭니다. 실무나 프로젝트에서 기술 선택을 검증하기 위한 테스트는 어느 수준까지 하는 게 적절한가요? 모든 기술 스택을 비교 검증하는 게 아니라면, 어떤 기술은 테스트로 검증하고 어떤 기술은 요구사항·팀 역량·생태계 등의 근거만으로 선택해도 되는지 그 기준이 궁금합니다.</p>

<p><b>Q2-1.</b><br>
redis 방식이 훨씬 더 정합성과 성능이 뛰어난 것을 알고 있었지만 낙관적 락, 비관적 락, redisson-rlock 등을 테스트하여 가장 성능이 좋은 걸 확인하고 redis-lua 방식을 채택했습니다. 실무에서는 이런 방식이 어떤 의미를 가지는지, 유의미한 과정으로 남기려면 어떻게 해야할지 궁금합니다.</p>

<p><b>Q2-2.</b><br>
저희는 비관적 락, 낙관적 락, DB 원자적 업데이트, Redisson RLock, Redis WATCH, Redis Lua를 비교해 대규모 동시 요청에서는 Redis Lua가 가장 높은 처리 성능을 보이는 것을 확인했습니다. 하지만 요청량과 경합이 적은 환경에서는 DB 원자적 업데이트만으로도 충분히 정합성을 보장할 수 있고, Redis를 추가하면 인프라 비용과 장애 지점, Redis–DB 동기화 문제가 늘어날 수 있다고 생각합니다. 따라서 Redis Lua가 항상 더 좋은 방식이라기보다, DB 락 대기와 커넥션 사용량이 일정 수준을 넘을 때 의미가 있는 선택이라고 이해했습니다. 실무에서는 다음 중 어떤 지표와 기준으로 DB 방식에서 Redis 방식으로 전환할 필요성을 판단하나요?</p>

<table>
  <thead>
    <tr><th>기준</th></tr>
  </thead>
  <tbody>
    <tr><td>동시 요청 수와 초당 요청 수</td></tr>
    <tr><td>DB 락 대기시간</td></tr>
    <tr><td>DB 커넥션 풀 대기 수</td></tr>
    <tr><td>p95 응답시간</td></tr>
    <tr><td>오류·타임아웃 비율</td></tr>
    <tr><td>DB CPU 사용률</td></tr>
    <tr><td>Redis 도입·운영 비용</td></tr>
  </tbody>
</table>

<p>또한 이번 실험을 단순히 가장 빠른 기술을 선정한 결과가 아니라, 각 방식이 처리할 수 있는 트래픽 범위와 Redis가 필요해지는 전환 지점을 찾는 과정으로 남기려면 어떤 조건으로 추가 측정하는 것이 적절한지 궁금합니다.</p>

<p><b>Q3.</b><br>
Redis에서 재고 확정 후 Kafka로 비동기 DB 적재하는 구조에서, Consumer의 DB 적재가 실패하면 이미 유저에게 성공 응답이 나간 뒤라 Redis 보상이 무의미해지는데, 실무에서는 이런 상황을 보통 어떻게 처리하나요?</p>

<p><b>Q4.</b><br>
저희 프로젝트에서 Redis 확정 이후 DB 적재 과정의 정합성과 동시성 사이의 트레이드 오프 문제를 고민하다가, 다음과 같은 방향으로 정리했습니다. 이 판단이 맞는지 검증받고 싶습니다.</p>
<ul>
  <li>일반적인 DB 아웃박스 패턴 대신, Redis Streams를 활용하기로 했습니다. 재고 확정(Lua 스크립트) 안에서 <code>XADD</code>로 이벤트도 함께 기록하면, 재고 차감과 이벤트 발행이 같은 원자적 연산으로 묶여서, 별도 DB 테이블(outbox)이나 Kafka 발행처럼 "확정과 기록 사이에 틈"이 생기는 문제를 피할 수 있고 동시에 Redis의 빠른 처리 속도도 챙길 수 있다고 판단했습니다. Consumer가 이 스트림을 읽어 DB에 비동기로 적재하고, 처리 완료는 <code>XACK</code>로 추적합니다.</li>
</ul>

<p><b>Q5.</b><br>
기술 스택이나 성능 테스트가 비슷해질 수밖에 없는 상황에서, 기능·설계·검증 방식·운영 관점 중 어떤 부분을 차별화 포인트로 가져가는 게 좋을까요?</p>

</details>
<details>	
   <summary>멘토링 질문 (8월 25일)</summary>
</details>
