# 가상 스레드 10,000개의 OS 스레드는 몇 개인가

> **실험일**: 2026-06-18 | **JDK**: Amazon Corretto 21.0.6 | **시리즈**: JVM 가상 스레드 실증 실험 #1

---

"가상 스레드는 OS 스레드가 아니라 JVM 힙 위의 객체다."

문서에서 읽은 문장입니다. 읽었을 때 고개를 끄덕였지만, 실제로 어느 정도인지는 몰랐습니다. 가상 스레드 10,000개를 띄우면 OS 스레드가 몇 개 생길까요? 네이티브 스택은 얼마를 씁니까? JVM 힙은 얼마나 늘어납니까?

추론으로 답하는 대신 직접 측정했습니다.

---

## 가설

측정 전에 검증할 명제를 세 개 뽑았습니다.

- **A-1**: 가상 스레드의 Thread/stack 사용량이 플랫폼 대비 1/100 이하다
- **A-2**: Java 힙 사용량은 가상 스레드 쪽이 더 크다 (Continuation이 힙에 올라가니까)
- **A-3**: OS 스레드 수가 가상 쪽에서 훨씬 적다

---

## 실험 환경

| 항목 | 값 |
|------|----|
| JDK | Amazon Corretto 21.0.6 |
| OS | macOS (kern.maxprocperuid = 2,666) |
| 플랫폼 스레드 수 | 1,500개 (`-Xss256k -Xmx512m`) |
| 가상 스레드 수 | 10,000개 (`-Xmx512m`) |
| 스레드 상태 | 전부 `CountDownLatch.await()` blocking |
| 측정 도구 | NMT(`jcmd VM.native_memory`), `ps -M`, `MemoryMXBean` |

플랫폼 스레드를 1,500개로 제한한 이유가 있습니다. macOS는 프로세스당 OS 스레드를 최대 2,666개로 제한합니다. JVM 내부 스레드가 이미 수십 개 차지하고 있어서 앱 레벨에서 1,500개가 현실적인 상한이었습니다. 가상 스레드는 10,000개를 띄워도 OS 스레드를 거의 안 쓰니 제한에 걸리지 않습니다.

---

## 실측 결과

### 핵심 수치

```
플랫폼 스레드 1,500개
  생성 시간:       80ms
  OS 스레드 수:    1,532개
  NMT stack:      441,712 KB (committed)
  힙 증가량:       +4,288 KB (per-thread 2.9 KB)

가상 스레드 10,000개
  생성 시간:       20ms
  OS 스레드 수:    40개
  NMT stack:      72,320 KB (committed)
  힙 증가량:       +23,915 KB (per-vthread 2.4 KB)
```

### Per-thread 정규화

| 항목 | 플랫폼 | 가상 | 배율 |
|------|-------|------|------|
| 네이티브 스택 (per-thread) | **278.7 KB** | **3.5 KB** | 80x 차이 |
| Java 힙 (per-thread) | 2.9 KB | 2.4 KB | 비슷 |
| 총 메모리 | ~281.6 KB | ~5.9 KB | ~48x |
| 생성 시간 | 80ms / 1,500개 | 20ms / 10,000개 | 4배 빠름 |
| OS 스레드 수 | 1,532 | **40** | 38x 적음 |

---

## 세 가지 발견

### 발견 1: 가상 스레드는 네이티브 스택에 0 KB를 씁니다 (A-1 확인)

플랫폼 스레드 1개는 생성될 때 OS에 네이티브 스택을 예약합니다. `-Xss256k` 설정 기준으로 스레드 1개당 약 278 KB입니다. 1,500개면 441 MB가 예약됩니다.

가상 스레드는 다릅니다. 10,000개를 만들어도 NMT Thread/stack이 크게 늘지 않습니다. 측정 결과에서 늘어난 72 MB는 가상 스레드 자체가 아니라 JVM이 자동으로 추가한 캐리어 스레드 17개의 스택입니다. 가상 스레드 10,000개의 네이티브 스택 기여분은 사실상 0입니다.

가상 스레드의 실행 상태(콜스택 프레임)는 네이티브 스택이 아니라 Java 힙 위의 Continuation 객체에 저장됩니다. OS가 아닌 JVM이 관리합니다.

### 발견 2: Java 힙 사용량은 플랫폼과 비슷합니다 (A-2 수정)

"가상 스레드는 Continuation이 힙에 올라가니까 힙을 더 쓴다"고 예상했는데, 틀렸습니다.

단순 blocking 상태(`CountDownLatch.await()`)에서 per-vthread 힙 비용은 **2.4 KB**였습니다. 플랫폼(2.9 KB)보다 오히려 약간 적습니다.

이유는 스택 깊이입니다. `await()` 상태에서 Continuation은 프레임이 거의 없어서 힙을 적게 씁니다. 깊은 콜스택을 가진 실행 중인 가상 스레드는 다릅니다. 실험 #7에서 콜스택 깊이가 힙에 미치는 영향을 따로 측정합니다.

### 발견 3: OS 스레드 수가 38배 적습니다 (A-3 확인)

가상 스레드 10,000개가 OS 스레드 40개로 처리됐습니다. 캐리어 스레드 ~17개와 JVM 내부 스레드 ~23개입니다. 앱이 만든 가상 스레드 10,000개는 OS 스레드를 단 한 개도 추가하지 않았습니다.

OS 스레드 수가 적다는 것은 컨텍스트 스위칭 비용, 메모리 사용량, OS 스케줄러 부하가 모두 낮다는 뜻입니다.

---

## 메모리 구조 비교

```
플랫폼 스레드 1개                      가상 스레드 1개
┌─────────────────────────────┐        ┌──────────────────────────────┐
│ OS Thread (pthread)         │        │ Java Heap 객체               │
│  └─ 네이티브 스택: 278 KB    │        │  └─ Thread 객체              │
│  └─ Thread 객체: 2.9 KB     │        │  └─ Continuation (스택 프레임)│
│                             │        │  총 ~2-3 KB                   │
│ 총 ~281 KB / thread         │        │                              │
│ OS 스레드 1:1 대응            │        │ 캐리어 스레드 공유             │
└─────────────────────────────┘        └──────────────────────────────┘
```

같은 자원으로 가상 스레드는 플랫폼 스레드 대비 약 **48~80배** 더 많은 스레드를 유지할 수 있습니다.

---

## 그렇다면 무한정 늘리면 되는가

가상 스레드가 OS 스레드 병목을 없앴다면, 스레드를 최대한 많이 만들면 처리량도 비례해서 오를까요?

실험 #2에서 이 질문에 답합니다. 스레드를 400개로 늘렸을 때 p50은 12ms였고 p99는 4,704ms였습니다. DB 커넥션 풀이 가상 스레드의 새로운 천장이 됩니다.

---

## 재현 방법

```bash
javac NMTProbe.java HeapUsageProbe.java

# 플랫폼 스레드 측정
java -Xss256k -XX:NativeMemoryTracking=detail -Xmx512m \
  NMTProbe platform 1500 nmt-platform.txt

# 가상 스레드 측정
java -XX:NativeMemoryTracking=detail -Xmx512m \
  NMTProbe virtual 10000 nmt-virtual.txt

# 힙 사용량 직접 측정
java -Xss256k -Xmx512m HeapUsageProbe platform 1500
java -Xmx512m HeapUsageProbe virtual 10000
```

실험 코드: [kmando01/jvm-virtual-threads-bench](https://github.com/kmando01/jvm-virtual-threads-bench)

---

## 시리즈 목록

| # | 제목 | 핵심 발견 |
|---|------|-----------|
| **1** | **NMT 메모리 분리** | **OS 스레드 38배 절약, per-thread 78배 경량** |
| 2 | [HikariCP 풀 사이즈 곡선](./2026-06-29-virtual-threads-hikaricp.md) | 커넥션 풀이 새 천장, p50은 거짓말한다 |
| 3 | [synchronized vs JNI Pinning](./2026-06-29-virtual-threads-pinning.md) | JEP 491이 고친 것과 고치지 못한 것 |
| 4 | [Continuation Heap과 ThreadLocal](./2026-06-29-virtual-threads-memory.md) | 콜스택 깊이와 ThreadLocal이 힙에 미치는 영향 |
