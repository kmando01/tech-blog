# 핫 도큐먼트는 자기만 느린 게 아니다

> **실험일**: 2026-06-29 | **스택**: MongoDB 8.0 / Go 1.24 | **시리즈**: 분산 시스템 실증 실험

---

단일 문서에 쓰기가 집중되면 그 문서 처리가 느려진다는 건 예상할 수 있습니다. 그런데 핫 도큐먼트를 한 번도 건드리지 않은 작업이 throughput -55%, p99 +120%를 경험한다면 어떨까요?

MongoDB write ticket pool이 공유 자원이기 때문입니다. 핫 도큐먼트가 ticket을 독식하면 무관한 작업도 대기 줄에 쌓입니다. MongoDB 8.0에서 write ticket이 128개에서 17개로 줄어들면서 이 현상이 더 예리하게 드러납니다.

---

## 배경: MongoDB write ticket pool

MongoDB는 동시 쓰기를 제어하기 위해 write ticket pool을 운영합니다. 쓰기 작업은 ticket을 하나 획득한 뒤 실행하고 반환합니다. ticket이 모두 소진되면 새 쓰기는 대기 줄에 쌓입니다.

MongoDB 버전별로 ticket 수가 달라졌습니다.

```
MongoDB 7.x 이전: 128개 (고정)
MongoDB 8.0:      ~17개 (CPU 코어 수 기반, 12코어 × 1.4 ≈ 17)
```

ticket 수가 128개에서 17개로 줄었습니다. 핫 도큐먼트가 ticket을 점유하면 남은 여유분이 훨씬 적습니다.

---

## 실험 설계

세 가지 시나리오를 workers 10→50→100→200→500 단계로 측정했습니다.

```
Hot    — 1개 문서에 모든 쓰기 집중 ($inc counter on "_id": "x")
Spread — 1,000개 문서에 분산 쓰기 (균등 분포)
Mixed  — 200 hot goroutine + 200 cold goroutine 동시 실행
```

Mixed 시나리오가 핵심입니다. hot goroutine은 단일 문서에, cold goroutine은 완전히 다른 1,000개 문서에 씁니다. hot document와 무관한 cold goroutine이 얼마나 피해를 입는지 측정합니다.

정합성도 함께 확인했습니다. 모든 실험에서 MongoDB OCC(Optimistic Concurrency Control) 재시도로 데이터 손실이 없었습니다.

---

## 실측 결과

### Throughput (ops/s) — Hot vs Spread

| workers | Spread | Hot | Hot 비교 |
|---------|--------|-----|---------|
| 10 | 29,826 | 29,887 | ≈ 동일 |
| 50 | 30,754 | 23,943 | **-22%** |
| 100 | 27,329 | 24,725 | -10% |
| 200 | 27,393 | **31,700** | **+16%** (역전!) |
| 500 | 27,432 | **32,046** | **+17%** (역전!) |

### P99 Latency (ms)

| workers | Spread | Hot | Hot 비교 |
|---------|--------|-----|---------|
| 10 | 0.99 | 1.47 | +48% |
| 50 | 6.59 | 11.63 | **+76%** |
| 100 | 16.14 | 19.84 | +23% |
| 200 | 32.29 | 26.83 | -17% |
| 500 | 77.70 | 68.74 | -11% |

### writeConflicts

| workers | Spread | Hot | 배율 |
|---------|--------|-----|------|
| 10 | 113 | 3,596 | **32x** |
| 50 | 453 | 26,881 | **59x** |
| 100 | 1,249 | 47,673 | **38x** |
| 200 | 1,827 | 98,772 | **54x** |
| 500 | 6,063 | 276,596 | **46x** |

### Write Queue Peak

| workers | Spread | Hot |
|---------|--------|-----|
| 10 | 0 | 0 |
| 50 | 0 | **31** |
| 100 | 48 | **82** |
| 200 | 17 | **183** |
| 500 | 0 | **411** |

---

## 세 가지 발견

### 발견 1: 중부하에서 핫 도큐먼트가 더 느리고, 고부하에서는 더 빠릅니다

workers=50에서 Hot TPS(23,943)는 Spread(30,754)보다 22% 낮습니다. 그런데 workers=200에서는 Hot TPS(31,700)가 Spread(27,393)보다 16% **높습니다**.

반직관적입니다. 이유가 있습니다.

```
Hot doc — 동일한 문서가 항상 WiredTiger buffer cache에 상주
           → 매 접근마다 cache hit 보장
           → page lookup 비용 없음

Spread — 1,000개 문서 무작위 접근
          → 다양한 page 접근 필요
          → cache miss 발생 가능
```

고부하에서 Hot doc은 "혼자는 빠릅니다." 하지만 동시에 write ticket을 독식합니다. workers=500에서 Hot의 peak write_tickets_out은 19/19, **100% 고갈**입니다.

### 발견 2: write ticket 100% 고갈이 핵심 신호입니다

Hot workers=50부터 이미 `peak_write_tickets_out = 19 / 18`로, total보다 많이 측정됩니다(50ms 폴링 타이밍 편차로 인한 일시적 초과). ticket이 사실상 포화 상태라는 신호입니다.

```
Hot w50:  peak tickets_out = 19 / 18  (한도 초과)
Hot w100: peak tickets_out = 18 / 18  (100% 점유)
Hot w200: peak tickets_out = 19 / 17  (한도 초과)
Hot w500: peak tickets_out = 19 / 19  (100% 점유)
```

ticket이 모두 소진되면 새 쓰기는 대기 줄에 쌓입니다. Hot w500의 peak write queue = **411**. 이 대기 줄이 hot과 무관한 작업도 막습니다.

### 발견 3: cold worker가 핫 도큐먼트 때문에 피해를 입습니다 (Mixed 시나리오)

이 발견이 가장 중요합니다. 혹시 "우리 코드에서 핫 도큐먼트를 다루는 부분은 분리돼 있으니 다른 서비스엔 영향 없다"고 생각하고 계신가요?

Mixed 시나리오 결과입니다.

```
설정: hot goroutine 200개 + cold goroutine 200개 (총 400개)
      hot   → 단일 문서 (_id: "x")
      cold  → 1,000개 분산 문서 (hot doc와 무관)

결과:
  Cold worker throughput: 12,200 ops/s
  순수 Spread w200 기준: 27,393 ops/s
  차이: -55%  ←  cold worker가 한 번도 hot doc을 건드리지 않았는데

  Cold worker p99: 71.17 ms
  순수 Spread w200 기준: 32.29 ms
  차이: +120%

  peak write_queue_length: 294
  순수 Spread w200 기준: 17
  차이: +17x
```

writeConflicts를 확인하면 57,747건이 모두 hot worker에서 발생했습니다. cold worker는 writeConflict가 0건임에도 throughput -55%, p99 +120%를 경험했습니다.

원인은 write ticket pool 공유입니다. hot goroutine들이 writeConflict 재시도를 반복하며 ticket을 점유하는 동안 cold goroutine들도 ticket을 기다려야 합니다.

---

## MongoDB 8.0의 write ticket 감소가 이 현상을 심화시킵니다

MongoDB 7.x까지는 write ticket이 128개였습니다. 8.0에서 CPU 코어 기반으로 바뀌면서 12코어 환경에서 17개가 됩니다.

17개라는 숫자가 의미하는 것: **17개 이상의 동시 쓰기가 핫 doc에 집중되면 즉시 ticket 포화**가 발생합니다. workers=50 시나리오에서 이미 한도를 초과했습니다.

과거 128개 환경이었다면 hot doc에 50개 goroutine이 집중돼도 ticket 여유가 많아 다른 작업에 영향이 작았을 것입니다. 8.0 환경에서는 workers=50에 이미 전체 시스템에 영향이 나타납니다.

---

## 운영 대응

### 발견 즉시 확인할 지표

```
metrics.operation.writeConflicts     — 급등 시 hot doc 신호
queues.execution.write.queueLength   — 17 초과 시 ticket 포화 중
queues.execution.write.out / total   — 비율이 높으면 ticket 부족
```

### 설계 대응

| 상황 | 대응 |
|------|------|
| 단일 카운터 집중 쓰기 | 버킷 샤딩 (N개 카운터 → 합산) |
| 재고 감소 등 원자적 연산 | Redis DECR + 이벤트 기반 MongoDB 동기화 |
| 집계 카운터 | 배치 집계로 전환 |

---

## 정리

핫 도큐먼트 문제는 해당 문서의 성능 문제가 아닙니다. write ticket pool이라는 공유 자원을 통해 DB 전체로 전파됩니다.

이 실험에서 확인한 것:
- writeConflicts가 0건인 cold worker도 throughput -55%, p99 +120% 피해를 입었습니다
- MongoDB 8.0에서 write ticket이 17개로 줄어 더 적은 동시 요청에도 포화가 발생합니다
- hot doc가 고부하에서 처리량이 높아 보이는 현상은 cache 상주 효과이며, 동시에 ticket 독식으로 다른 작업을 희생시킵니다

`metrics.operation.writeConflicts` 가 급등하면 단순히 "특정 문서가 느리다"가 아니라 "DB 전체 쓰기가 영향받을 수 있다"고 판단해야 합니다.

---

## 재현 방법

```bash
cd hot-doc-bench/go-bench
go build -o go-bench .

# 단계별 비교
./go-bench ladder 500

# Collateral damage 실증
./go-bench mixed 400 500

# 개별 실행
./go-bench spread 200 500
./go-bench hot 200 500
```

실험 코드: [kmando01/mongodb-hot-document-bench](https://github.com/kmando01/mongodb-hot-document-bench)
