# 우리가 본 게 진짜였을까 — MongoDB 트랜잭션의 connection 함정

> **실험일**: 2026-07-01 | **환경**: Spring Boot 3.3.4 + mongo-java-driver 5.x + MongoDB 7.0 / Atlas Replica Set | **시리즈**: 분산 시스템 실증 실험

트랜잭션 코드를 짜다 보면 한 번쯤 이런 의문이 듭니다. **이 트랜잭션, connection에 묶여 있는 건가?**

JDBC에 익숙하신 분일수록 자연스럽게 떠오르는 질문입니다. JDBC에서는 `Connection`이 트랜잭션을 소유합니다. `setAutoCommit(false)` → `commit()`까지 한 connection 위에서 일어나고, connection이 다른 곳으로 가면 트랜잭션도 따라가지 않습니다.

MongoDB는 같은 모델일까요? 답을 찾으려고 로그를 켰는데, 거기서 **잘못된 결론으로 너무 쉽게 빠지는 함정**을 발견했습니다. 이 글은 그 함정과, 그걸 어떻게 빠져나왔는지의 기록입니다.

## 1. 함정의 시작 — 너무 자연스러운 첫인상

평범하게 트랜잭션을 돌리고 로그를 봤습니다. Spring Boot + Atlas Replica Set 환경입니다.

```kotlin
writeTx.execute {
    mongoTemplate.save(doc1, col)   // command logged: conn=7
    mongoTemplate.save(doc2, col)   // command logged: conn=7
    mongoTemplate.save(doc3, col)   // command logged: conn=7
}
```

세 명령 모두 `conn=7`입니다. 자연스러운 결론은 **"역시 JDBC처럼 connection에 묶이는구나"**였습니다. 한 트랜잭션 = 한 connection. 이대로 글을 닫고 코드로 돌아갔다면, 이 멘탈 모델을 그대로 가지고 살았을 겁니다.

혹시 같은 로그를 보고 같은 결론을 내리셨을까요?

여기서 멈추지 않은 이유는 단순했습니다. **"내가 본 게 진짜로 그 의미인지"**가 갑자기 의심스러워졌습니다.

같은 관찰("`conn=7`이 세 번")이 가리킬 수 있는 진실은 적어도 두 가지입니다.

- **진실 A** — 트랜잭션이 connection을 *소유*한다. 그래서 같은 connection만 쓰인다.
- **진실 B** — 트랜잭션은 connection과 *무관*하다. 어떤 이유로 같은 connection이 반복 선택될 뿐이다.

둘은 표면 결과가 완전히 똑같습니다. 그리고 평소 워크로드에서는 둘 중 무엇이 참인지 *알 방법이 없습니다*. 이게 함정의 정체입니다. **관찰이 두 가설을 구분하지 못하면, 저희는 그냥 직관(JDBC 멘탈 모델)에 맞는 쪽으로 결론을 내리고 맙니다.**

진실을 알려면 두 가설이 갈라지는 지점을 찾아야 했습니다.

## 2. 명세는 이미 답을 갖고 있다 — 하지만 명세만으로는 부족하다

MongoDB Specifications의 `transactions.md`를 봤습니다. 트랜잭션을 어떻게 식별하는지에 대한 답이 있었습니다.

모든 명령에 두 필드가 함께 전송됩니다.

- `lsid` — logical session id (UUID)
- `txnNumber` — 그 세션 안에서 단조 증가하는 번호

> "Drivers MUST increment the `txnNumber` for the corresponding server session."

서버는 이 둘의 조합으로 트랜잭션을 인식합니다. **명세 어디에도 connection이 트랜잭션 식별에 사용된다는 진술은 없습니다.**

명세상으로는 진실 B입니다. 그런데 *명세가 그렇게 쓰여 있다*는 것과 *실제로 그렇게 동작한다*는 건 별개입니다. 드라이버 구현이 명세를 어겼을 수도, 환경의 어떤 레이어가 추가 제약을 걸고 있을 수도 있습니다. 명세 한 줄로 "B가 답"이라고 단정하면 그건 *읽은 것을 믿은* 거지 *검증한* 게 아닙니다.

명세는 가설을 강하게 만들었습니다. 하지만 결정적 증거는 아니었습니다.

## 3. 드라이버 코드 — 같은 connection이 *강제*가 아니라는 단서

다음 단계는 드라이버였습니다. `mongo-java-driver`의 `ClientSessionBinding.java`를 직접 봤습니다.

```java
public Connection getConnection(final OperationContext operationContext) {
    TransactionContext<Connection> transactionContext = TransactionContext.get(session);
    if (transactionContext != null && transactionContext.isConnectionPinningRequired()) {
        // ... pin된 connection 반환
    } else {
        return wrapped.getConnection(operationContext);  // ★ 풀에서 그때그때
    }
}
```

`TransactionContext`는 같은 파일에서 다음 조건일 때만 만들어집니다.

```java
if (clusterType == SHARDED || clusterType == LOAD_BALANCED) {   // ★
    TransactionContext<Connection> transactionContext = new TransactionContext<>(clusterType);
    ...
}
```

**저희 환경(단일 Replica Set)은 이 분기에 걸리지 않습니다.** TransactionContext가 만들어지지 않으니 connection pinning 코드 자체가 실행되지 않습니다. `getConnection`은 매번 풀에서 그냥 가져옵니다.

그렇다면 저희가 본 "같은 connection 반복"은 *드라이버의 강제*가 아닙니다. 그럼 뭘까요? `ConcurrentPool` 코드에서 단서를 찾았습니다.

```java
// available.addLast(conn)   ← 반납: tail에 추가
// available.pollLast()      ← 체크아웃: tail에서 꺼냄 = LIFO
```

**Pool이 LIFO(Last In First Out)입니다.** 가장 최근 반납된 connection이 다음 체크아웃에서 가장 먼저 나옵니다. 단일 스레드로 명령을 순차 실행하면, 명령마다 같은 connection이 LIFO top으로 계속 올라옵니다.

이로써 1절의 관찰은 새로운 해석을 얻습니다. **저희가 본 것은 "트랜잭션이 connection을 쥐고 있다"가 아니라 "pool이 LIFO다"였습니다.** 두 현상이 단일 스레드 워크로드에서는 완전히 똑같이 관찰됩니다.

그래도 *그럴듯함*은 증명이 아닙니다. 진실 A가 여전히 완전히 죽지는 않았습니다. *결정적으로* 갈라내려면 한 단계가 더 필요했습니다.

## 4. 명제를 깨려고 한다 — 평소엔 안 보이는 시나리오를 강제로 만들기

여기서 추리 방식을 바꿨습니다. **명제를 지지하는 관찰을 더 모으는 게 아니라, 명제가 거짓이라면 일어나야 할 일을 직접 발생시켜봤습니다.**

진실 A가 참이라면 — 트랜잭션이 connection을 소유한다면 — 한 트랜잭션의 명령은 *반드시* 같은 connection으로만 인식되어야 합니다. 다른 connection으로 같은 `lsid+txnNumber`를 보내면 서버가 "이 connection엔 그런 트랜잭션 없음"이라며 거부해야 합니다.

진실 B가 참이라면 — 트랜잭션이 connection과 무관하다면 — 다른 connection으로 보내도 서버는 `lsid+txnNumber`만 보고 한 트랜잭션으로 처리해야 합니다.

이걸 직접 보면 갈립니다. 문제는 평소엔 LIFO 때문에 항상 같은 connection만 쓰이니, 그 상황 자체가 발생하지 않는다는 점입니다. **그래서 LIFO를 인위적으로 깼습니다.**

세 단계로 실험했습니다.

### 단계 1 — Pool이 정말 LIFO인지 확인 (H-0)

가설의 전제부터 확인했습니다. 동시 트랜잭션 3개로 pool에 connection 3개(`7, 8, 9`)를 강제 생성한 뒤, 순차 트랜잭션 5회를 돌렸습니다.

```text
── warmup: 동시 TX 3개 ──
update warm-0 | conn=7
update warm-1 | conn=8
update warm-2 | conn=9

── 순차 TX 5회 ──
update seq-0 | conn=9
update seq-1 | conn=9
update seq-2 | conn=9
update seq-3 | conn=9
update seq-4 | conn=9
```

Pool에 3개나 있는데 순차는 전부 `conn=9`입니다. LIFO 확정입니다.

### 단계 2 — 같은 connection 위에 별개 트랜잭션 (H-1)

`conn=9` 하나로 두 트랜잭션을 통과시켜봤습니다.

```text
update h1-tx1 | lsid: JkLI420a... txnNumber=7 | conn=9
commitTx     | lsid: JkLI420a... txnNumber=7 | conn=9
update h1-tx2 | lsid: JkLI420a... txnNumber=8 | conn=9
commitTx     | lsid: JkLI420a... txnNumber=8 | conn=9
```

`conn=9` 하나로 `txnNumber=7`과 `txnNumber=8` 두 트랜잭션이 처리됐습니다. **같은 connection이라도 트랜잭션은 별개입니다.** 약한 증거지만 첫 균열이었습니다.

### 단계 3 — 결정적 실험: 한 트랜잭션을 여러 connection 위에 흘려보내다 (H-2)

이게 두 가설을 결정적으로 갈라냅니다.

설계는 단순합니다. 메인 트랜잭션의 insert들 사이에 `Thread.sleep(100ms)`를 끼우고, 그 사이에 background 15개 스레드가 트랜잭션을 돌리며 pool을 휘젓습니다. 15개 스레드가 동시에 connection을 열고 닫으면서 pool이 기존 7·8·9를 넘어 확장되고, LIFO top이 매번 달라지므로 메인 트랜잭션의 다음 명령은 다른 connection을 받게 됩니다.

```kotlin
val stop = AtomicBoolean(false)

writeTx.execute {  // ← 하나의 트랜잭션 (하나의 lsid)
    mongoTemplate.save(doc("main-0"), col)

    repeat(15) { i ->
        Thread {
            while (!stop.get()) {
                writeTx.execute { mongoTemplate.save(doc("bg-$i"), col) }
            }
        }.start()
    }

    repeat(4) { idx ->
        Thread.sleep(100)  // 이 사이에 background가 pool 흔듦
        mongoTemplate.save(doc("main-${idx + 1}"), col)
    }

    stop.set(true)  // background 스레드 중단
}

// 커밋 후 원자성 검증: main-0 ~ main-4 모두 존재해야 함
val count = mongoTemplate.count(Query(), col)
check(count == 5L) { "expected 5 docs, got $count" }
```

결과입니다.

| insert | lsid | txnNumber | autocommit:false | conn |
| --- | --- | --- | --- | --- |
| main-0 | oUTClqKa.. | 1 | ✓ | **9** |
| main-1 | oUTClqKa.. | 1 | ✓ | **8** ← 바뀜 |
| main-2 | oUTClqKa.. | 1 | ✓ | **25** ← 바뀜 |
| main-3 | oUTClqKa.. | 1 | ✓ | **56** ← 바뀜 |
| main-4 | oUTClqKa.. | 1 | ✓ | **56** |

다섯 번의 insert가 **네 개의 서로 다른 connection**을 통해 나갔습니다. 그런데 `lsid`와 `txnNumber`는 그대로입니다. **서버는 이 모든 명령을 하나의 트랜잭션으로 받아들였고, 커밋 후 doc 5개가 모두 존재함을 조회로 확인했습니다.**

진실 A가 참이었다면 거부됐어야 합니다. 거부되지 않았습니다. **진실 B가 답입니다.**

## 5. 그래서 답은 — connection 단위가 아니다

- **트랜잭션 경계를 정의하는 것은 wire protocol에 박힌 `lsid + txnNumber`입니다.**
- **Connection은 그 정보를 실어 나르는 TCP 파이프일 뿐입니다.** 파이프가 도중에 바뀌어도 서버는 신경 쓰지 않습니다.

| | JDBC | MongoDB |
| --- | --- | --- |
| 트랜잭션 식별 | Connection 자체 | `lsid + txnNumber` |
| 다른 connection으로 같은 트랜잭션 명령? | 개념적으로 불가능 | 가능 (서버는 식별자만 봄) |

여기까지가 *기술적* 결론입니다. 그런데 이 글에서 더 가져가고 싶은 건 다른 결론입니다.

## 6. 진짜 교훈 — 평소엔 두 가지 진실이 똑같이 보인다

이 추적이 의미 있었던 건 답을 얻어서가 아닙니다. **"내가 본 것을 어떻게 해석할 것인가"**라는 질문이 답보다 더 무겁다는 걸 깨달아서입니다.

1절로 돌아가 보겠습니다. 저희는 `conn=7`이 세 번 나오는 로그를 봤습니다. 이 관찰은 진실 A(connection 단위)와 진실 B(`lsid` 단위) 둘 다와 양립합니다. 그리고 평소 워크로드에서는 — 단일 스레드, 짧은 트랜잭션, 적당한 부하 — **이 둘은 영원히 똑같이 보입니다**.

이게 무서운 부분입니다. 저희는 **단일 스레드 환경의 LIFO가 만들어준 거짓 안정성** 위에서 멘탈 모델을 만듭니다. 그 모델이 진실 A든 B든, 결과가 같으니까 알아챌 일이 없습니다. 어느 날 동시성이 올라가거나, 다른 환경(샤드, LB)에 배포되거나, 트랜잭션이 길어지면, 그제서야 "어, 왜 안 되지?" 하게 됩니다.

두 가지 교훈을 가져갑니다.

**(a) 지지하는 관찰은 약하다. 깨려는 실험이 강하다.**

`conn=7`이 세 번 나온 관찰을 백 번 더 모아도 진실 A와 B를 구분할 수 없습니다. *명제를 지지하는 증거*는 우연일 수 있습니다. 진짜 검증은 *명제가 거짓이라면 일어나야 할 일*을 강제로 일으켜보는 것입니다. H-2의 실험 설계가 그것입니다. "LIFO를 깨면 어떻게 되지?"를 물은 순간, 두 진실이 처음으로 갈라졌습니다.

**(b) 안 보이는 것을 보이게 만드는 일이 디버깅의 본질이다.**

평소엔 진실 B의 동작(다른 connection을 가로지르는 트랜잭션)이 *발현되지 않습니다*. 발현되지 않으니 측정할 수 없고, 측정할 수 없으니 검증할 수 없습니다. H-2가 한 일은 background 15개 스레드로 pool을 흔들어 **잠재된 동작을 강제로 발현시킨 것**입니다. 동시성을 강제로 끌어올린 그 순간이, 이 글 전체에서 가장 의미 있는 한 줄이었습니다.

이건 MongoDB 트랜잭션 이야기를 넘습니다. **자기 코드의 동작을 진짜로 알고 있다는 것은, 그 동작이 *평소 워크로드에서 발현되지 않는 시나리오*에서 어떻게 되는지까지 측정해본 상태를 말합니다.** "잘 동작한다"고 믿는 코드 중 다수는, 사실은 "*평소 워크로드에서는* 잘 동작한다"이고, 그 둘은 같지 않습니다.

처음 `conn=7`이 세 번 나온 로그를 본 순간, 그것을 "JDBC식 connection 단위로 동작하는 증거"로 받아들이고 끝냈다면 이 글은 없었을 겁니다. **관찰을 의심하는 능력 — 그게 같은 로그를 본 두 개발자를 갈라놓는 차이입니다.** 코드를 검증할 때, 지지하는 증거를 찾으시나요, 아니면 깨려는 실험을 설계하시나요?

## 부록 — 환경별 동작 정리

| 배포 형태 | Connection 동작 | 강제성 |
| --- | --- | --- |
| **단일 Replica Set** | 명세상 pin 없음. Pool LIFO로 같은 connection이 자주 쓰일 뿐 | 강제 아님 |
| **Sharded Cluster** | 첫 명령의 mongos에 ClientSession을 pin (server pin) | MUST |
| **Load Balanced** | Connection 자체에 pin | MUST |

세 경우 모두 **트랜잭션의 식별자는 여전히 `lsid + txnNumber`**입니다. Pinning은 라우팅 정합성을 위한 구현 디테일이지, 트랜잭션 식별 자체가 connection으로 바뀌는 게 아닙니다.

## 참고 자료

- [MongoDB Specifications — transactions.md](https://github.com/mongodb/specifications/blob/master/source/transactions/transactions.md)
- [MongoDB Specifications — load-balancers.md](https://github.com/mongodb/specifications/blob/master/source/load-balancers/load-balancers.md)
- [mongo-java-driver — ClientSessionBinding.java](https://github.com/mongodb/mongo-java-driver/blob/main/driver-sync/src/main/com/mongodb/client/internal/ClientSessionBinding.java)
- [MongoDB Manual — Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
