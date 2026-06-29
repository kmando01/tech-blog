# 기술 블로그

실험으로 검증한 기술 이야기를 씁니다. 추론 사슬을 만들고, 가설을 세우고, 수치로 확인합니다.

## 시리즈

### JVM 가상 스레드 실증 실험

가상 스레드를 도입하기 전에 알아야 할 것들을 직접 측정했습니다.  
메모리 구조부터 HikariCP 병목, synchronized/JNI pinning, Continuation 힙 비용까지.

| # | 제목 | 핵심 발견 |
|---|------|-----------|
| 1 | [가상 스레드 10,000개의 OS 스레드는 몇 개인가](./posts/2026-06-18-virtual-threads-nmt.md) | OS 스레드 38배 절약, per-thread 78배 경량 |
| 2 | [가상 스레드 400개를 켰을 때 p99에 무슨 일이 생기는가](./posts/2026-06-29-virtual-threads-hikaricp.md) | 커넥션 풀이 새 천장, p50은 거짓말한다 |
| 3 | [JEP 491이 고친 것과 고치지 못한 것](./posts/2026-06-29-virtual-threads-pinning.md) | synchronized 해결, JNI pinning은 JDK 25에서도 잔존 |
| 4 | [가상 스레드 10만 개를 띄우기 전에 계산해야 할 두 가지](./posts/2026-06-29-virtual-threads-memory.md) | Continuation 힙 비용, ThreadLocal × vthread = OOM |

---

실험 코드: [kmando01/jvm-virtual-threads-bench](https://github.com/kmando01/jvm-virtual-threads-bench)
