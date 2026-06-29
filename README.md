# 기술 블로그

실험으로 검증한 기술 이야기를 씁니다. 추론 사슬을 만들고, 가설을 세우고, 수치로 확인합니다.

## 시리즈

### JVM 가상 스레드 실증 실험

가상 스레드를 도입하기 전에 알아야 할 것들을 직접 측정했습니다. 메모리 구조부터 HikariCP 병목, synchronized/JNI pinning, CPU-bound 한계까지.

| 글 | 핵심 발견 |
|----|-----------|
| [가상 스레드 400개를 켰을 때 p99에 무슨 일이 생기는가](./posts/2026-06-29-virtual-threads-hikaricp.md) | 커넥션 풀이 새 천장, p50은 거짓말한다 |

---

실험 코드: [kmando01/jvm-virtual-threads-bench](https://github.com/kmando01/jvm-virtual-threads-bench)
