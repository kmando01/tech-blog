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

### 분산 시스템 실증 실험

Redis와 MongoDB에서 "성능을 높이려다 오히려 낮아진" 사례들을 수치로 검증했습니다.

| # | 제목 | 핵심 발견 |
|---|------|-----------|
| 1 | [Redis 분산락을 달았더니 처리량이 16배 떨어졌다](./posts/2026-06-29-redis-distributed-lock.md) | 수학적 TPS 상한, writeConflict 제거의 역설 |
| 2 | [MongoDB replaceOne vs $set: 예측이 두 번 빗나갔다](./posts/2026-06-29-mongodb-replace-vs-set.md) | dirty bytes 1.9x(예측 33x), oplog 175x(예측 33x) |
| 3 | [핫 도큐먼트는 자기만 느린 게 아니다](./posts/2026-06-29-mongodb-hot-document.md) | cold worker -55% 피해, write ticket pool 인질 |

---

실험 코드: [kmando01/jvm-virtual-threads-bench](https://github.com/kmando01/jvm-virtual-threads-bench)
