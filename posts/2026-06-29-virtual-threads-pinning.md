# JEP 491이 고친 것과 고치지 못한 것 — synchronized vs JNI Pinning

> **실험일**: 2026-06-18 | **JDK**: Amazon Corretto 21.0.6 / 25.0.3 | **시리즈**: JVM 가상 스레드 실증 실험 #3

---

JDK 25에서 JEP 491이 정식 포함됐습니다. `synchronized` 블록 안에서 가상 스레드가 blocking될 때 캐리어 스레드에 고정(pinned)되는 문제를 해결한 것입니다.

그렇다면 "JDK 25로 올리면 pinning 걱정은 끝"인가요?

반은 맞고 반은 틀립니다. `synchronized` pinning은 완전히 해결됐습니다. 그런데 JNI 호출 안에서의 pinning은 JDK 25에서도 그대로입니다. JDK 21에서 4.10배 느린 것이 JDK 25에서 4.04배 느립니다. 사실상 변화가 없습니다.

이 글은 그 차이를 JDK 21/25에서 직접 측정한 과정입니다.

---

## 배경: Pinning이 왜 문제인가

가상 스레드의 핵심 장점은 blocking I/O가 발생했을 때 캐리어 스레드를 반환하고 다른 가상 스레드에게 양보하는 것입니다.

```
일반적인 가상 스레드 동작:
  가상 스레드 → Thread.sleep(50ms) 호출
  → JVM: "캐리어 반환, 다른 가상 스레드를 실행"
  → 50ms 후 재개
  → 50개가 동시에 sleep 중이어도 캐리어 12개로 모두 처리
```

그런데 `synchronized` 블록 안에서 blocking이 발생하면 다릅니다.

```
JDK 21 pinning 상황:
  가상 스레드 → synchronized(obj) 진입 → Thread.sleep(50ms)
  → JVM: "monitor를 쥔 채 unmount하면 다른 스레드가 monitor를 잡을 수 없음"
  → 판정: 캐리어 스레드 유지 (pinned)
  → 12코어 기준 캐리어 12개 포화 → 나머지 38개 가상 스레드 대기
```

결과적으로 동시 실행 가능한 가상 스레드 수가 캐리어 스레드 수(기본 = CPU 코어 수)로 제한됩니다.

---

## 실험 설계

### 핵심 설계 원칙

pinning과 lock 경합을 격리해야 합니다. **가상 스레드마다 자신만의 lock 객체를 보유**하면 lock 경합 없이 pinning 효과만 측정할 수 있습니다.

```java
// 각 가상 스레드가 독립적인 lock 보유 — 경합 없음
Object lock = new Object(); // 스레드별 생성

synchronized (lock) {
    Thread.sleep(50); // blocking 발생
}
```

이렇게 하면 "synchronized가 느린 이유가 경합인가, pinning인가"를 분리할 수 있습니다.

### 이론 예측

| 시나리오 | 예측 시간 | 근거 |
|----------|----------|------|
| pinning 없음 | ~200ms | ops=200, concurrency=50 → 4라운드 × 50ms |
| pinning 있음 | ~850ms | 실질 병렬도 = 12코어 → 17라운드 × 50ms |

### 실험 환경

| 항목 | 값 |
|------|----|
| JDK | 21.0.6 vs 25.0.3 (Amazon Corretto) |
| 동시 가상 스레드 | 50개 |
| blocking 시간 | 50ms |
| 총 ops | 200 |
| CPU 코어 | 12 (= 캐리어 스레드 기본 풀) |

---

## 실험 C: synchronized Pinning — JDK 21 vs 25

### 실측 결과

```
=== JDK 21 ===
이론 최소 (no pinning): 200ms
이론 최악 (pinning):    850ms

[synchronized]   913ms  throughput=219 ops/s   ← 이론 최악에 근접
[ReentrantLock]  211ms  throughput=948 ops/s   ← 이론 최소에 근접

RatioSync/Lock = 4.33x → PINNING CONFIRMED

=== JDK 25 ===
이론 최소 (no pinning): 200ms
이론 최악 (pinning):    850ms

[synchronized]   221ms  throughput=905 ops/s   ← 이론 최소에 근접
[ReentrantLock]  219ms  throughput=913 ops/s

RatioSync/Lock = 1.01x → NO PINNING (JEP 491 효과)
```

### 수치 정리

| JDK | synchronized | ReentrantLock | 비율 | 결론 |
|-----|-------------|---------------|------|------|
| 21 | 913ms / 219 ops/s | 211ms / 948 ops/s | **4.33x** | Pinning 확인 |
| 25 | 221ms / 905 ops/s | 219ms / 913 ops/s | **1.01x** | Pinning 해결 |

JDK 25에서 `synchronized`와 `ReentrantLock`의 차이가 1% 이내로 줄었습니다. 완전히 동일한 처리량입니다.

### JEP 491이 한 일

JDK 21에서 monitor는 캐리어 스레드에 연결돼 있었습니다. unmount하면 monitor를 쥔 채 캐리어가 사라지므로 다른 스레드가 monitor를 잡지 못합니다. 그래서 캐리어를 유지했습니다.

JDK 25에서 JEP 491은 monitor를 가상 스레드에 연결했습니다. 이제 가상 스레드가 unmount돼도 monitor는 가상 스레드에 묶여 있습니다. 캐리어는 자유롭게 다른 가상 스레드를 실행할 수 있습니다.

---

## 실험 D: JNI Pinning — JDK 21 vs 25

`synchronized`가 해결됐으니, JNI는 어떨까요?

JNI(Java Native Interface)는 C 라이브러리를 자바에서 호출하는 방법입니다. JDBC 드라이버 일부, SSL/TLS 구현, 암호화 라이브러리 등이 JNI를 씁니다. JNI 함수 안에서 blocking이 발생하면 어떻게 됩니까?

### 실측 결과

```
=== JDK 21 ===
이론 최소 (no pin): 200ms | 이론 최악 (pin): 850ms

[Java sleep]   222ms  throughput=901 ops/s   ← 정상 (no pin)
[JNI  sleep]   911ms  throughput=220 ops/s   ← 이론 최악에 근접

RatioJNI/Java = 4.10x → JNI PINNING CONFIRMED

=== JDK 25 (JEP 491 이후에도 JNI pinning 잔존) ===
이론 최소 (no pin): 200ms | 이론 최악 (pin): 850ms

[Java sleep]   226ms  throughput=885 ops/s   ← 정상 (no pin)
[JNI  sleep]   914ms  throughput=219 ops/s   ← 이론 최악에 근접

RatioJNI/Java = 4.04x → JNI PINNING CONFIRMED
```

### JDK 21 vs 25 JNI 비교

| JDK | JNI sleep | Java sleep | 비율 | 결론 |
|-----|----------|-----------|------|------|
| 21 | 911ms / 220 ops/s | 222ms / 901 ops/s | **4.10x** | Pinning |
| 25 | 914ms / 219 ops/s | 226ms / 885 ops/s | **4.04x** | **여전히 Pinning** |

4.10x와 4.04x의 차이는 JDK 25의 전반적인 스케줄러 개선에 의한 노이즈 수준입니다. 실질적으로 동일합니다.

### 왜 JNI는 고칠 수 없는가

`synchronized`는 JVM이 monitor 소유권을 관리하기 때문에 JVM 수준에서 고칠 수 있었습니다. JNI는 다릅니다.

```
JNI 호출 흐름:
  가상 스레드 → nativeSleep() JNI 호출
  → C 스택 프레임 진입
  → OS: usleep(50,000μs) — pthread를 sleep
  → pthread = 캐리어 스레드 = OS가 관리하는 실제 스레드
  → JVM이 "이 캐리어가 native에서 sleep 중"임을 알지만
     pthread를 깨워서 다른 일을 시킬 수 없음 (OS sleep이 끝날 때까지)
```

native frame이 실행 중인 동안 C 코드의 스택 포인터, TLS(Thread-Local Storage), CPU 레지스터가 OS 스레드에 묶여 있습니다. JVM이 끼어들 수 없는 OS 수준의 제약입니다.

| | synchronized | JNI |
|--|------------|-----|
| pinning 원인 | JVM monitor가 캐리어에 연결 | native thread stack이 OS에 연결 |
| fix 방법 | monitor를 vthread에 연결 (JEP 491) | **불가 (OS 수준 제약)** |
| JDK 21 | pinning | pinning |
| JDK 25 | **해결** | **여전히 pinning** |

---

## 실험 F: Carrier Thread 튜닝으로 JNI 처리량 회복하기

JNI pinning이 근본적으로 해결 안 된다면, 완화책은 무엇입니까?

캐리어 스레드 수를 늘리면 됩니다. JNI로 인해 캐리어가 줄어든 상황에서 캐리어를 더 공급하는 것입니다.

### 실측 결과

```
parallelism=12    JNI=1,927ms(  208 ops/s)  Java=  69ms(5,797 ops/s)
parallelism=25    JNI=  934ms(  428 ops/s)  Java=  69ms(5,797 ops/s)
parallelism=50    JNI=  474ms(  844 ops/s)  Java=  72ms(5,556 ops/s)
parallelism=100   JNI=  250ms(1,600 ops/s)  Java=  75ms(5,333 ops/s)
parallelism=200   JNI=  138ms(2,899 ops/s)  Java=  74ms(5,405 ops/s)
```

| parallelism | JNI(ms) | JNI ops/s | 이론(ms) | 실측/이론 |
|-------------|---------|----------|---------|---------|
| 12 (기본값) | 1,927 | 208 | 1,700 | ≈ 1.13x |
| 25 | 934 | 428 | 800 | ≈ 1.17x |
| 50 | 474 | 844 | 400 | ≈ 1.19x |
| 100 | 250 | 1,600 | 200 | ≈ 1.25x |
| 200 | 138 | 2,899 | 100 | ≈ 1.38x |

**2배 parallelism → 처리량 2배**: 이론값과 거의 선형으로 일치합니다.

### 설정 방법

```bash
-Djdk.virtualThreadScheduler.parallelism=100
```

완전 회복(JNI ≈ Java sleep)을 위해서는 `parallelism ≥ 동시 JNI blocking 작업 수`가 되어야 합니다. 이 실험에서 동시 JNI 작업이 400개였으므로 이론상 parallelism=400이 필요합니다.

### 이 방법의 한계

parallelism=200으로 올리면 OS 스레드가 200개 생깁니다. 기존 플랫폼 스레드풀과 같은 수준의 자원을 쓰는 셈입니다. 가상 스레드의 경량성 이점이 줄어듭니다.

더 나은 대안은 JNI 작업을 별도 플랫폼 스레드풀에 오프로드하는 것입니다.

```java
// JNI 작업을 별도 executor에서 실행
ExecutorService jniPool = Executors.newFixedThreadPool(poolSize);

// 가상 스레드는 결과만 기다림
CompletableFuture<Result> future = CompletableFuture.supplyAsync(
    () -> nativeOperation(),
    jniPool
);
Result result = future.get(); // 가상 스레드는 carrier를 반환하고 대기
```

이렇게 하면 JNI blocking이 플랫폼 스레드 안에서 일어나고, 가상 스레드는 carrier를 유지하지 않습니다.

---

## 실무 체크리스트

JDK 21 환경에서 pinning을 진단하는 방법입니다.

```bash
# JFR로 VirtualThreadPinned 이벤트 수집
java -XX:StartFlightRecording=filename=pin.jfr,settings=default MyApp

# 또는 실시간 스택 트레이스 출력
java -Djdk.tracePinnedThreads=full MyApp
```

| 사용 중인 코드 | JDK 21 | JDK 25 | 권고 |
|-------------|-------|-------|------|
| `synchronized` + blocking I/O | pinning ❌ | 해결 ✅ | JDK 25 업그레이드 |
| JDBC native 드라이버 | pinning ❌ | pinning ❌ | 오프로드 또는 parallelism 증가 |
| SSL/TLS 핸드셰이크 (native) | pinning 가능 ❌ | pinning 가능 ❌ | 확인 필요 |
| 짧은 native 계산 (암호화, 수학) | 낮은 위험 | 낮은 위험 | 10μs 이하면 무시 가능 |

---

## 이 실험에서 가장 놀라운 점

사전 예측은 "JNI blocking 시 에러가 난다"였습니다. 다른 언어 드라이버(Mongoose, .NET)에서는 트랜잭션 안에서 잘못된 설정을 하면 에러를 던집니다.

MongoDB Java 드라이버 silent override 실험에서 같은 패턴을 봤습니다. 에러가 없어서 문제를 알아채기 어려운 상황이 더 위험합니다.

JNI pinning도 마찬가지입니다. 에러가 없고 로그도 없습니다. `-Djdk.tracePinnedThreads=full` 없이는 운영 중에 발견하기 어렵습니다.

---

## 재현 방법

```bash
# C JNI 라이브러리 빌드
JAVA_HOME=$(/usr/libexec/java_home -v 21)
clang -shared -fPIC -I"$JAVA_HOME/include" -I"$JAVA_HOME/include/darwin" \
  -o libjni_sleep.dylib jni_sleep.c

javac PinningProbe.java

# JDK 21
java PinningProbe

# JDK 25
JAVA25=~/vt-jdk25/amazon-corretto-25.jdk/Contents/Home/bin/java
$JAVA25 -cp out25 PinningProbe 2>/dev/null
```

실험 코드: [kmando01/jvm-virtual-threads-bench](https://github.com/kmando01/jvm-virtual-threads-bench)

---

## 시리즈 목록

| # | 제목 | 핵심 발견 |
|---|------|-----------|
| 1 | [NMT 메모리 분리](./2026-06-18-virtual-threads-nmt.md) | OS 스레드 38배 절약, per-thread 78배 경량 |
| 2 | [HikariCP 풀 사이즈 곡선](./2026-06-29-virtual-threads-hikaricp.md) | 커넥션 풀이 새 천장, p50은 거짓말한다 |
| **3** | **synchronized vs JNI Pinning** | **JEP 491이 고친 것과 고치지 못한 것** |
| 4 | [Continuation Heap과 ThreadLocal](./2026-06-29-virtual-threads-memory.md) | 콜스택 깊이와 ThreadLocal이 힙에 미치는 영향 |
