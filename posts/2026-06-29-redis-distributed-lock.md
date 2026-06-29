# Redis 분산락을 달았더니 처리량이 16배 떨어졌다

> **실험일**: 2026-06-29 | **스택**: Go 1.26 / MongoDB 8.0 / Redis 8.0 | **시리즈**: 분산 시스템 실증 실험

---

선착순 쿠폰 발급 서버에 Redis 분산락을 달았습니다. writeConflict는 0이 됐습니다. 처리량은 28,491 TPS에서 1,725 TPS로 떨어졌습니다.

락을 추가해서 성능이 16배 낮아진 것입니다. 그리고 이 결과는 실측 전에 수식으로 예측할 수 있었습니다.

---

## 배경: 왜 분산락을 고려했는가

선착순 쿠폰 발급에서 재고가 0인데 발급되는 문제가 생겼습니다. MongoDB의 단일 문서에 `remaining` 필드를 두고, 여러 인스턴스가 동시에 감소시키는 구조에서 writeConflict가 누적되면서 일부 요청이 잘못된 시점의 값을 읽고 있었습니다.

흔히 선택하는 해결책이 Redis 분산락입니다. "락으로 한 번에 하나씩 처리하면 race condition이 없어진다." 이 판단이 맞는지 측정하기 전에 먼저 수식으로 따져봤습니다.

---

## 이론 예측: 분산락의 처리량 상한

분산락은 모든 요청을 직렬화합니다. 동시에 단 하나의 요청만 처리됩니다. 이 구조에서 TPS의 수학적 상한은 다음과 같습니다.

```
최대 TPS = 1 / lock_hold_time
```

락 보유 시간이 0.55ms라면 초당 처리 가능한 최대 요청 수는 1,818건입니다. 동시 요청이 100개든 1,000개든 상관없습니다. 어차피 한 번에 하나만 처리되니까요.

P99 latency도 수식으로 예측할 수 있습니다. polling 방식(10ms 간격으로 락 획득 재시도)을 쓴다면 대기 순서 끝에 선 요청의 지연이 P99에 나타납니다.

```
P99 ≈ (workers - 1) × polling_delay
```

workers=500이면 P99 ≈ 499 × 10ms = 4,990ms.

이 예측이 실측과 얼마나 맞는지 확인했습니다.

---

## 실험 설계

세 가지 시나리오를 비교했습니다.

| 시나리오 | 락 | 문서 구조 |
|---------|----|----|
| A — 베이스라인 | 없음 | 단일 hot document (`remaining` 필드) |
| B — 분산락 | Redis SETNX 폴링 (10ms) | 단일 hot document |
| C — 분산 설계 | 없음 | Redis DECR + 쿠폰 번호별 개별 문서 |

A와 B는 같은 MongoDB 구조에서 락 유무만 다릅니다. C는 근본적으로 다른 설계입니다. C를 넣은 이유는 "락이 아닌 다른 선택지가 있는가"를 함께 확인하기 위해서입니다.

```
실험 환경
  MongoDB 8.0.10 (standalone), Redis 8.0, Go goroutine 기반
  ops/worker: 20, 총 재고: 100,000
  workers: 10 → 50 → 100 → 200 → 500 (Ladder)
  락 TTL: 500ms, 폴링 간격: 10ms
```

Python threading은 GIL 때문에 진정한 동시 실행이 안 됩니다. 락 경합을 정확히 측정하려면 Go goroutine이 필요했습니다.

---

## 실측 결과

### Throughput (TPS)

| workers | A (락 없음, hot) | B (분산락) | C (분산 설계) |
|---------|---------------|-----------|-------------|
| 10 | 15,311 | 1,912 | 13,002 |
| 50 | 29,806 | 1,986 | 22,934 |
| 100 | 28,491 | **1,725** | **36,777** |
| 200 | 29,558 | 1,836 | 36,246 |
| 500 | 37,747 | 1,835 | 39,827 |

### P99 Latency (ms)

| workers | A | B | C |
|---------|---|---|---|
| 10 | 3 | 76 | 3 |
| 50 | 2 | 389 | 12 |
| 100 | 11 | **864** | **5** |
| 200 | 12 | 1,622 | 15 |
| 500 | 29 | **2,919** | 22 |

### WriteConflicts

| workers | A | B | C |
|---------|---|---|---|
| 100 | 4,672 | **0** | **0** |
| 500 | 23,932 | 0 | 0 |

---

## 세 가지 발견

### 발견 1: 분산락의 TPS는 workers와 무관하게 상수에 고착됩니다

B의 처리량을 보면 workers가 10→50→100→200→500으로 50배 늘어도 TPS는 1,912 → 1,986 → 1,725 → 1,836 → 1,835, **사실상 ~1,800에 고착**됩니다.

이것은 이상 현상이 아닙니다. 수식이 예측한 그대로입니다.

```
이론 최대 TPS = 1 / 0.55ms = 1,818
실측 평균 TPS = ~1,800
```

동시 요청이 아무리 많아도 초당 처리할 수 있는 한계가 lock_hold_time에 묶여 있습니다. workers를 10배 늘려도 처리량은 1%도 오르지 않습니다.

혹시 지금 "workers를 늘리면 처리량이 오를 것"이라고 기대하며 분산락 환경에서 인스턴스를 추가하고 있지는 않으신가요? 인스턴스가 늘면 P99만 오릅니다. TPS 상한은 바뀌지 않습니다.

### 발견 2: P99는 workers에 비례해 폭증합니다

```
workers=10:   P99=76ms   (≈ 9 × 10ms)
workers=50:   P99=389ms  (≈ 49 × 10ms)
workers=100:  P99=864ms  (≈ 99 × 10ms)
workers=200:  P99=1,622ms (≈ 199 × 10ms)
workers=500:  P99=2,919ms (≈ 499 × 10ms)
```

이론값(499 × 10ms = 4,990ms)보다 실측이 낮은 이유는 polling이 일부 miss되기 때문입니다. 패턴 자체는 workers에 선형 비례합니다.

P99=864ms일 때 "평균 응답시간"을 보면 괜찮아 보일 수 있습니다. workers=100 기준으로 99명 중 98명은 빠르게 처리되고 1명이 864ms를 기다리는 구조이기 때문입니다. **평균이 정상이어도 꼬리는 폭발합니다.**

### 발견 3: writeConflict는 사라졌지만 문제도 사라진 게 아닙니다

B는 writeConflict=0을 달성했습니다. 그런데 A의 writeConflict는 문제였을까요?

```
A (workers=500): writeConflicts=23,932, TPS=37,747
B (workers=500): writeConflicts=0,       TPS=1,835
```

MongoDB는 writeConflict가 발생하면 내부적으로 재시도합니다. WiredTiger write ticket pool을 활용해 다른 요청을 병렬 처리하면서 conflict된 요청을 재시도합니다. A는 conflict가 많아도 TPS가 37,000까지 올라갑니다.

분산락은 이 병렬성을 없앴습니다. conflict는 0이지만 처리량도 1/20 수준이 됐습니다. "conflict를 없애야 한다"는 목표는 달성했지만, 그 방법이 처리량을 더 크게 희생시켰습니다.

---

## C가 모든 지표에서 이기는 이유

C는 근본적으로 다른 접근입니다. 단일 hot document를 쟁탈하는 대신, 재고 확보와 문서 생성을 분리했습니다.

```go
// C 패턴 핵심
remaining := redis.DECR("remaining")  // 원자적 재고 확보, 락 불필요
if remaining < 0 { return "매진" }

serialNo := redis.INCR("issued")      // 쿠폰 번호 발급
mongo.UpdateOne(
    {serial_no: serialNo},            // 각자 독립 문서 — hot doc 없음
    {$set: {...}}, {upsert: true}
)
```

Redis `DECR`은 단일 명령으로 원자적입니다. 락 없이 race condition을 방지합니다. MongoDB 문서는 쿠폰 번호별로 독립적이어서 서로 충돌하지 않습니다.

결과: workers=500에서 39,827 TPS, P99 22ms. B(1,835 TPS, 2,919ms) 대비 TPS 22배, P99 132배 차이입니다.

---

## 분산락을 써도 되는 경우

이 실험이 "분산락은 나쁘다"는 결론은 아닙니다. 처리량 상한을 이해하고 쓰면 됩니다.

```
단순 재고 관리 (처리량 < 1,000 TPS 목표)
  → 분산락 가능. lock_hold_time 역수가 TPS 상한임을 인지.

고처리량 선착순 발급 (> 10,000 TPS 목표)
  → C 패턴 (Redis DECR + 분산 document) 필수.
```

분산락을 쓴다면 아래 설정이 중요합니다.

```go
lockTTL        = 100ms   // 짧게 — hang 시 자동 해제
lockRetryDelay = 1ms     // polling 대신 pub/sub 알림 방식 권장
lockTimeout    = 3s      // SLA 기준으로 설정
```

Redisson의 pub/sub 방식을 쓰면 polling latency는 줄일 수 있지만 **TPS 상한(1/lock_hold_time)은 변하지 않습니다.**

---

## 모니터링 기준

분산락 환경에서 알람을 설정한다면 이 지표를 봐야 합니다.

| 지표 | 경고 기준 | 의미 |
|------|---------|------|
| TPS가 일정 값에 고착 | 기대 TPS의 1/5 이하 | 직렬화 포화 — 분산 설계로 전환 필요 |
| lock_wait P99 | > 500ms | 락 대기 큐 누적 중 |
| MongoDB writeConflicts/s | > 1,000 | hot document 경합 발생 |

이 실험을 마치고 남은 질문이 생겼습니다. "MongoDB hot document 자체가 문제라면, 락 없이도 cold worker가 얼마나 피해를 입는가?" 다음 글에서 그 측정을 다룹니다.

---

## 재현 방법

```bash
# MongoDB 8.x, Redis 8.x, Go 1.22+ 필요
mongod --dbpath /tmp/mongodb-data --port 27017 &
redis-server --port 6379 &

cd go-bench
go run main.go
# 결과: results/ladder.json
```

실험 코드: [kmando01/redis-distributed-lock-bench](https://github.com/kmando01/redis-distributed-lock-bench)
