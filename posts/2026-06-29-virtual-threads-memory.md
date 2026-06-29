# 가상 스레드 10만 개를 띄우기 전에 계산해야 할 두 가지

> **실험일**: 2026-06-18 | **JDK**: Amazon Corretto 21.0.6 / 25.0.3 | **시리즈**: JVM 가상 스레드 실증 실험 #4

---

가상 스레드 10만 개를 띄운다고 가정해봤습니다. 메모리가 얼마나 필요할까요?

"가상 스레드는 가볍다"는 말은 맞습니다. 하지만 가볍다는 게 무료라는 뜻은 아닙니다. 가상 스레드의 실행 상태는 JVM 힙에 올라가는 Continuation 객체에 저장됩니다. 그리고 기존 코드에서 ThreadLocal을 쓰고 있다면, vthread 수만큼 그 페이로드가 힙에 곱해집니다.

두 가지를 직접 측정했습니다.

---

## 실험 G: Continuation 스택 깊이 vs 힙 비용

### 왜 스택 깊이가 힙에 영향을 미치는가

플랫폼 스레드는 실행 상태(콜스택 프레임)를 네이티브 스택에 저장합니다. 스택 깊이가 늘어나도 힙은 영향을 받지 않습니다. 깊이와 무관하게 `-Xss`로 지정한 전체 크기를 미리 예약합니다.

가상 스레드는 다릅니다. 콜스택 프레임이 Continuation 객체로 직렬화되어 힙에 저장됩니다. blocking이 발생하면 JVM이 현재 스택 프레임을 힙으로 옮기고 캐리어를 반환합니다. 이 때 스택 깊이가 깊을수록 Continuation 크기가 커집니다.

### 실험 설계

가상 스레드 5,000개를 동시에 띄우고, 각 스레드의 콜스택 깊이를 1에서 500까지 바꾸면서 힙 사용량을 측정했습니다. 각 프레임에는 지역 변수 `long a, b` (16 bytes)를 뒀습니다.

모든 가상 스레드는 `CountDownLatch.await()`으로 blocking 상태를 유지했습니다. GC 3회 강제 후 `MemoryMXBean`으로 힙 사용량을 측정했습니다.

### 실측 결과

```
=== Experiment G: Continuation Stack Depth vs Heap ===
virtual threads per run: 5,000

depth    heap_delta(KB)    per_thread(B)
----------------------------------------
1        13,116            2,686
10       15,801            3,236
50       15,033            3,078     ← depth 10보다 낮음 (JIT 효과)
100      23,865            4,887
200      42,999            8,806
500      103,894           21,277
```

| depth | per_thread(B) | baseline 대비 |
|-------|-------------|-------------|
| 1 | 2,686 | — |
| 10 | 3,236 | +20% |
| 100 | 4,887 | +82% |
| 200 | 8,806 | +228% |
| 500 | 21,277 | +692% |

depth 50이 depth 10보다 낮게 측정된 이유가 있습니다. JIT 컴파일러가 해당 구간에서 인라이닝이나 escape analysis 최적화를 적용했기 때문입니다. depth 100 이상부터는 JIT 한계를 넘어 Continuation 직렬화 비용이 그대로 드러납니다.

### 실무에서 스택 깊이가 문제가 되는 경우

Spring MVC에서 요청 하나가 처리될 때 콜스택을 세어보면 놀랄 수 있습니다. Dispatcher Servlet → Interceptor → AOP Advisor → Service → Repository 체인을 거치면 쉽게 100~200 프레임이 됩니다.

| 상황 | per_thread | 10만 vthread 총 비용 |
|------|------------|-------------------|
| 단순 blocking (depth ~10) | ~3 KB | **~300 MB** |
| 일반 웹 요청 (depth ~100) | ~5 KB | **~500 MB** |
| Spring 복잡 요청 (depth ~200) | ~9 KB | **~900 MB** |
| 깊은 재귀 (depth 500) | ~21 KB | **~2.1 GB** |

10만 vthread, depth 200 기준으로 Continuation만 약 900 MB입니다. `-Xmx512m`으로는 턱없이 부족합니다.

---

## 실험 H: ThreadLocal × 가상 스레드의 힙 비용

### 문제의 구조

ThreadLocal은 스레드별로 독립적인 값을 저장하는 메커니즘입니다. `ThreadLocal<T>.get()`을 호출하면 현재 스레드의 `ThreadLocalMap`에서 값을 찾습니다. 스레드 수만큼 독립 복사본이 힙에 생깁니다.

가상 스레드도 스레드입니다. vthread마다 `ThreadLocalMap`이 생기고, 거기에 저장된 페이로드가 각각 힙에 올라갑니다.

```
ThreadLocal<byte[]> context = new ThreadLocal<>();

// vthread 10만 개가 각자 10KB를 set하면:
// 10만 × 10KB = 1GB 힙 사용
```

플랫폼 스레드였다면 수백 개만 동시에 운영하니 수 MB 수준입니다. 가상 스레드 10만 개를 쓰면 같은 코드가 1GB를 씁니다.

### 실험 설계

세 가지 컨텍스트 전파 방법을 비교했습니다.

- **ThreadLocal**: 스레드별 독립 복사본
- **InheritableThreadLocal**: 부모 스레드에서 자식으로 전파
- **ScopedValue** (JDK 21+, JEP 506): 불변 공유 바인딩

페이로드 크기(1KB, 10KB, 100KB)와 vthread 수(10,000, 100,000)를 바꾸면서 per-vthread 힙 비용을 측정했습니다.

### 실측 결과

| mode | payload | N=10K (KB/vt) | N=100K (KB/vt) |
|------|---------|--------------|----------------|
| 없음 (baseline) | 0 | 2.32 | 1.05 |
| **ThreadLocal** | 10KB | **12.43** | **11.44** |
| **ThreadLocal** | 100KB | **102.82** | **OOM** |
| InheritableThreadLocal | 10KB | 2.29 | 1.15 |
| ScopedValue | 100KB | **2.86** | 1.12 |

### 발견 1: ThreadLocal은 페이로드가 vthread 수만큼 곱해집니다

```
ThreadLocal 10KB × vthread 10,000개  → per-vt 12.43KB  (extra 10.11KB = 페이로드와 거의 같음)
ThreadLocal 10KB × vthread 100,000개 → per-vt 11.44KB  (extra 10.39KB)

ThreadLocal 100KB × vthread 10,000개  → per-vt 102.82KB
ThreadLocal 100KB × vthread 100,000개 → OutOfMemoryError (이론: 100KB × 10만 = 9.77GB > Xmx2g)
```

"ThreadLocal × 가상 스레드 = 메모리 시한폭탄"이라는 말이 과장이 아닙니다. 플랫폼 스레드 시대에는 동시 스레드 수가 수백 개라 페이로드가 작아도 됐습니다. 가상 스레드 10만 개 환경에서 같은 코드가 OOM을 냅니다.

### 발견 2: InheritableThreadLocal은 예상과 달리 거의 비용이 없습니다

처음 예상은 "ITL이 더 비싸다"였습니다. 실측은 정반대였습니다.

```
InheritableThreadLocal 10KB, N=10K: per-vt = 2.29KB ≈ baseline(2.32KB)
                                     → 페이로드 10KB인데 extra 비용 거의 0
```

이유는 Java의 `ITL.childValue()` 기본 구현이 `return parentValue`이기 때문입니다. 자식 vthread들이 부모의 `byte[]` 객체를 가리키는 **포인터만 복사**합니다. 페이로드 자체는 복제되지 않습니다.

```
[부모 스레드] itl → byte[10KB] ←──────────────┐
                                               │ 동일 객체
[vthread 0]  itl → ref ────────────────────►  byte[10KB]
[vthread 1]  itl → ref ────────────────────►  (같은 객체)
[vthread N]  itl → ref ────────────────────►  (같은 객체)
```

per-vthread 추가 비용은 `ThreadLocalMap.Entry` 메타데이터(포인터 하나)뿐입니다.

**중요한 예외**: `childValue()`를 오버라이드해서 deep copy를 반환하는 구현은 ThreadLocal과 같은 N배 힙 비용이 발생합니다. Logback의 MDC 같은 실무 라이브러리가 여기에 해당합니다.

### 발견 3: ScopedValue는 페이로드 크기와 무관합니다

혹시 지금 100KB짜리 ScopedValue와 1KB짜리 ScopedValue의 힙 비용 차이가 얼마일지 짐작이 가시나요?

```
ScopedValue 1KB,   N=10K: per-vt = 3.05KB
ScopedValue 10KB,  N=10K: per-vt = 3.01KB
ScopedValue 100KB, N=10K: per-vt = 2.86KB   ← 100KB인데 extra 비용 0.54KB만!
```

페이로드 크기가 1KB에서 100KB로 100배 늘었는데 per-vthread 비용 변화는 0.19KB입니다. ScopedValue는 불변 공유 바인딩입니다. `ScopedValue.where(sv, payload).run(...)` 실행 시 바인딩 레코드(`Carrier`)가 생성되지만 payload 자체는 복제되지 않습니다. N개의 vthread가 동일한 객체를 참조합니다.

---

## 세 모델 비교

```
ThreadLocal                    InheritableThreadLocal         ScopedValue
      │                               │                            │
  N개 독립 복사본              부모 값 공유 (ref copy)         공유 + 불변
      │                               │                            │
힙: N × payload               힙: ~0 추가                   힙: ~0 추가
Mutable: 가능                  Mutable: 자식 한정             Mutable: 불가
Thread-safe: X                 Thread-safe: X                 Thread-safe: 설계상 보장
```

| 시나리오 | 권장 |
|---------|------|
| 작업별 독립 mutable 상태 (MDC, 사용자 세션) | ThreadLocal — 단, vthread 수 × 페이로드 힙 예산 확보 필수 |
| 읽기 전용 컨텍스트 전파 (인증 정보, trace ID) | **ScopedValue** — 힙 비용 0, 불변 보장 |
| 부모→자식 전파, 읽기 위주 | InheritableThreadLocal — `childValue()` 오버라이드 여부 확인 필수 |

---

## 힙 예산 산정 공식

실험 G와 H를 합치면 vthread 도입 전에 계산해야 할 공식이 나옵니다.

```
총 힙 예산 = vthread 수 × (Continuation + ThreadLocal 페이로드) + 앱 기본 힙

Continuation ≈ 2KB + depth × 0.04KB  (G 실험 기준)
ThreadLocal  ≈ Σ(각 ThreadLocal 페이로드 크기)

예: 10만 vthread, 평균 스택 깊이 100, MDC 1KB 사용
  = 100,000 × (2 + 100×0.04 + 1) KB
  = 100,000 × 7 KB
  = 700 MB + 앱 기본 힙
  → -Xmx 최소 2GB 이상 권장
```

가상 스레드를 늘리기 전에 이 계산을 먼저 해보세요. p50이 정상이어도 힙 용량이 부족하면 GC 압박이 쌓이면서 p99가 서서히 올라가기 시작합니다.

---

## 재현 방법

```bash
# 실험 G: Continuation 깊이
javac ContinuationDepthProbe.java
java -Xmx1g ContinuationDepthProbe

# 실험 H: ThreadLocal 비용 (JDK 25 권장)
JAVA25=~/vt-jdk25/amazon-corretto-25.jdk/Contents/Home/bin/java
javac ThreadLocalCostProbe.java

# 기본 비교
$JAVA25 -Xmx2g ThreadLocalCostProbe none 0 10000
$JAVA25 -Xmx2g ThreadLocalCostProbe tl 10 10000
$JAVA25 -Xmx2g ThreadLocalCostProbe scoped 10 10000

# OOM 확인
$JAVA25 -Xmx2g -XX:+ExitOnOutOfMemoryError ThreadLocalCostProbe tl 100 100000
```

실험 코드: [kmando01/jvm-virtual-threads-bench](https://github.com/kmando01/jvm-virtual-threads-bench)

---

## 시리즈 마무리

4편에 걸쳐 가상 스레드의 천장을 하나씩 측정했습니다.

| # | 천장 | 측정값 | 해결 |
|---|------|--------|------|
| 1 | OS 스레드 | 플랫폼 1,500개 = OS 1,532개 vs 가상 10,000개 = OS 40개 | 가상 스레드가 해결 |
| 2 | DB 커넥션 풀 | pool=5, vthreads=400 → p99 4,704ms | pool size 재조정 |
| 3 | synchronized Pinning | 4.33x 처리량 손실 (JDK 21) | JDK 25 (JEP 491) |
| 3 | JNI Pinning | 4.04x 손실 (JDK 25에도 동일) | 오프로드 또는 parallelism 증가 |
| 4 | Continuation 힙 | depth 200: 9KB/vthread × 10만 = 900MB | Xmx 계획 수립 |
| 4 | ThreadLocal 힙 | 10KB × 10만 vthread = 1GB | ScopedValue로 전환 |

가상 스레드가 OS 스레드 병목을 없애면, 그다음 병목들이 수면 위로 올라옵니다. 각 천장을 미리 알고 있으면 도입 후 예상치 못한 상황을 줄일 수 있습니다.

---

## 시리즈 목록

| # | 제목 | 핵심 발견 |
|---|------|-----------|
| 1 | [NMT 메모리 분리](./2026-06-18-virtual-threads-nmt.md) | OS 스레드 38배 절약, per-thread 78배 경량 |
| 2 | [HikariCP 풀 사이즈 곡선](./2026-06-29-virtual-threads-hikaricp.md) | 커넥션 풀이 새 천장, p50은 거짓말한다 |
| 3 | [synchronized vs JNI Pinning](./2026-06-29-virtual-threads-pinning.md) | JEP 491이 고친 것과 고치지 못한 것 |
| **4** | **Continuation Heap과 ThreadLocal** | **콜스택 깊이와 ThreadLocal이 힙에 미치는 영향** |
