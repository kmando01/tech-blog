# MongoDB가 동시에 두 곳에서 무너지는 이유

도큐먼트 하나를 통째로 교체하는 코드가, 트래픽이 임계점을 넘는 순간 WiredTiger 캐시와 oplog 복제 레이어를 **동시에** 마비시킵니다. 두 레이어는 완전히 다른 코드 경로인데, 왜 같은 시점에 같이 무너질까요? 이 글은 그 둘을 잇는 단 하나의 변수를 공식 문서와 소스 README로 추적합니다.

---

## 배경: "큰 write"가 만드는 장애의 인상

MongoDB를 운영하다 보면 이런 장애 패턴을 한 번쯤 마주치게 됩니다. 평소에는 괜찮다가, 트래픽 피크 때만 동시에 두 가지 증상이 나타납니다.

- 쓰기 응답 시간이 기하급수적으로 늘어나고
- secondary lag이 동시에 치솟습니다

처음에는 "인프라 문제겠지"라고 생각하기 쉽습니다. RAM이 부족하거나, 네트워크 대역폭이 포화됐거나. 그런데 서버를 증설해도 임계점만 늦춰질 뿐, 패턴은 그대로 반복됩니다. 두 레이어가 **동시에** 무너진다는 게 핵심 단서입니다.

---

## 두 레이어를 잇는 단 하나의 변수

결론부터 말씀드리겠습니다.

> **Write 한 번의 payload size가 두 레이어를 연결하는 단일 변수입니다.**

캐시 측에서는 큰 payload가 dirty bytes를 빠르게 채워 eviction trigger를 돌파하고, 복제 측에서는 큰 payload가 oplog entry를 키워 16MB 한계를 넘기면 secondary가 멈춥니다. 두 경로 모두 payload size가 커질수록 나빠지기 때문에, 트래픽 임계점에서 동시에 터집니다.

RAM을 늘리거나 oplog 디스크를 확장하는 건 trigger 도달 시점을 늦출 뿐입니다. 근본 해결은 **payload 자체를 줄이는 것**입니다.

---

## 캐시 레이어: application thread가 청소부로 차출되는 순간

WiredTiger는 캐시 안의 dirty data(아직 디스크에 쓰이지 않은 변경분) 비율에 따라 두 단계로 반응합니다.

| 파라미터 | 기본값 | 동작 |
|---|---|---|
| `eviction_dirty_target` | 5% | background eviction thread 시작 |
| `eviction_dirty_trigger` | 20% | **application thread가 강제 참여** |

두 번째 단계가 문제입니다. WiredTiger 공식 문서는 이렇게 적고 있습니다.

> "Application threads will be throttled if the percentage of dirty data reaches the `eviction_dirty_trigger`."

application thread가 eviction에 강제 참여한다는 건, 클라이언트 요청을 처리해야 할 thread가 캐시 청소 작업에 차출된다는 뜻입니다. WiredTiger의 동시 write 처리 상한(write ticket)은 128개인데, 이 thread들이 청소 작업에 묶이면 ticket이 고갈되고, 요청 큐가 폭증하면서 응답 시간이 기하급수적으로 늘어납니다.

큰 도큐먼트를 통째로 교체하는 write는 이 dirty bytes를 빠르게 채웁니다. 트래픽이 낮을 때는 background eviction이 따라잡지만, 피크 때는 20% 선을 돌파합니다.

---

## 복제 레이어: secondary가 멈추는 두 가지 이유

### `$set`이 oplog에 더 작게 기록되는 이유

MongoDB `$set` 공식 문서:

> "Efficient Oplog Entries: `$set` optimizes replication by writing only the updated fields to the oplog instead of the entire document."

`replaceOne(doc)`으로 도큐먼트를 통째로 교체하면 도큐먼트 전체가 oplog entry에 기록됩니다. `updateOne({$set: {field: v}})`는 변경 필드만 diff로 기록합니다.

여기서 자주 빠지는 함정이 있습니다. **배열을 통째로 교체하는 `$set`은 회피가 안 됩니다.** `$set: {arr: [...]}` 형태는 배열 전체가 oplog에 인라인으로 들어갑니다. 배열 안 특정 element를 수정하는 위치 지정 연산자(`arr.$.field`, `arrayFilters`)를 써야 sub-document diff로 축약되어 oplog entry size가 줄어듭니다.

### 16MB 한계를 넘으면 chain으로 쪼개진다

MongoDB Transactions 공식 문서:

> "MongoDB creates as many oplog entries as necessary... each oplog entry still must be within the BSON document size limit of 16MB."

트랜잭션 크기가 16MB를 넘으면 commit 시점에 oplog entry가 `prevOpTime`으로 연결된 **chained applyOps** 형태로 분할됩니다.

그리고 Replication Source README(`mongo/src/mongo/db/repl/README.md`)에는 이 chain의 치명적인 속성이 기술되어 있습니다.

> "A secondary must wait until it receives the final `applyOps` oplog entry of a large unprepared transaction before applying entries."

chain의 첫 entry가 도착해도, **마지막 entry까지 전부 받기 전에는 아무것도 적용하지 않습니다.** 트랜잭션이 클수록 "첫 entry 도착 ~ 마지막 entry 도착" 사이 시간이 늘어나고, 이 구간이 secondary lag으로 관측됩니다.

### timestamp hole이 뒤따르는 모든 op을 차단한다

공식 문서에는 한 가지 더 있습니다.

> "If writeB commits first at Timestamp2, replication pauses until writeA commits, since writeA's oplog entry (Timestamp1) is required before replication can copy oplog entries to secondaries."

큰 트랜잭션이 이른 timestamp를 점유한 채 늦게 commit하면, 이후 들어온 작은 op들도 replication이 멈춥니다. slow query 로그의 `totalOplogSlotDurationMicros`로 이 구간을 측정할 수 있습니다.

---

## 흔한 오해: "secondary lag = secondary 멈춤"이 아닙니다

이 부분은 직접 검증해보기 전까지 오해하기 쉬운 지점입니다.

공식 문서는 이렇게 설명합니다.

> "Read operations that target secondaries... read from a WiredTiger snapshot of the data if the read takes place on a secondary where replication batches are being applied. Reading from a snapshot guarantees a consistent view of the data, and allows the read to occur simultaneously with the ongoing replication without the need for a lock."

secondary lag이 발생하는 상황에서도 secondary read는 멈추지 않습니다. 실제 동작은 **stale data 반환**입니다. 동일한 `_id`에 대한 write만 직렬 대기하고, 다른 `_id`의 write는 `replWriterThreadCount` 기반 thread pool로 병렬 적용됩니다.

---

## MongoDB 버전에 따라 달라지는 것들

### Oplog는 4.0+부터 설정 크기를 초과해 성장한다

Replica Set Oplog 공식 문서:

> "Unlike other capped collections, the oplog can grow past its configured size limit to avoid deleting the majority commit point."

4.0+ 이후 oplog 설정값은 최댓값이 아니라 **최솟값**처럼 동작합니다. secondary가 majority lag을 넘어 떨어지지 않도록 oplog가 보호됩니다. "oplog 윈도우 회복"보다 "oplog bloat 회복"이 정확한 표현입니다.

### 8.0의 writer/applier 분리

MongoDB 8.0 Release Notes:

> "Starting in MongoDB 8.0, secondaries write and apply oplog entries for each batch in parallel. A writer thread reads new entries from the primary and writes them to the local oplog. An applier thread asynchronously applies these changes to the local database."

| | 7.0 이하 | 8.0+ |
|---|---|---|
| 수신·적용 | 단일 thread | writer / applier 분리 |
| `w:majority` 반환 | 과반이 **applied** | 과반이 **written (received)** |
| 측정 지표 | `metrics.repl.buffer.sizeBytes` | `.write.sizeBytes` / `.apply.sizeBytes` |

8.0에서는 chained applyOps 메커니즘 자체가 약화될 수 있습니다. 6.x/7.x 환경과 8.0+ 환경을 같은 기준으로 비교하면 안 됩니다.

---

## 실전 적용 가이드

### Write 패턴 결정 기준

**`$set` 부분 업데이트를 써야 하는 상황**
- 변경 필드가 도큐먼트 전체의 일부일 때 (대부분의 update 케이스)
- ORM/ODM이 dirty tracking을 자동으로 해주지 않는 환경
- 트랜잭션 내부 update — oplog payload 최소화가 곧 chained applyOps 회피

**full replace가 의도적으로 옳은 상황**
- 도큐먼트 전체가 의미적으로 한 단위로 갈아끼워질 때 (캐시 entry 통째 교체)
- 변경 필드가 너무 많아 `$set` 명시가 오히려 더 복잡할 때
- 마이그레이션·일괄 보정 작업 (maintenance window를 잡고 진행)

### 배열 갱신 패턴

| 패턴 | oplog 영향 |
|---|---|
| `$set: {arr: <newArr>}` | 배열 전체 인라인 — 회피 필요 |
| `arr.$.field` | 단일 매칭 element만 diff |
| `arr.$[id].field` + `arrayFilters` | 조건 매칭 다수 element diff |
| `$push`, `$pull`, `$addToSet` | 추가/제거 연산만 기록 |

### 모니터링 지표

평균 기반 알람은 p99 long-tail에 둔감합니다. 아래 지표는 p99/max 기반으로 별도 알람을 설정해두는 게 좋습니다.

| 지표 | 조회 방법 | 임계 |
|---|---|---|
| WiredTiger dirty % | `serverStatus().wiredTiger.cache["tracked dirty bytes"] / ["maximum bytes configured"]` | 5% → background, 20% → application thread 차출 |
| Oplog entry max size | `db.oplog.rs.aggregate([{$sample:...}, {$project:{sz:{$bsonSize:"$$ROOT"}}}])` | 16MB 근접 시 chained 임박 |
| Flow control | `serverStatus().flowControl.isLagged` | `true`면 primary throttled |
| (8.0+) Apply buffer | `metrics.repl.buffer.apply.sizeBytes` | write 안정 + apply 증가 → applier 병목 |

---

## 마무리

두 레이어가 동시에 무너지는 것처럼 보였던 이유는, 결국 payload size라는 하나의 변수가 두 경로를 동시에 압박하기 때문이었습니다. 인프라 증설은 trigger 도달 시점만 미룰 뿐, 패턴 자체를 바꾸지 않습니다.

ORM의 `save()`나 `replaceOne(doc)` 같은 자동 전체 교체 패턴을 팀 코드베이스에서 찾아보는 것부터 시작해보시면 좋겠습니다. 특히 배열 필드를 포함한 도큐먼트를 다루는 코드가 있다면, `$set: {arr: [...]}` 형태가 있는지 확인해보시길 권합니다.

아직 남아 있는 질문이 하나 있습니다. MVCC long-running transaction이 dirty bytes에 미치는 영향, 그리고 partial index의 인덱스 페이지 dirty write는 이 글에서 다룬 경로와는 별개의 메커니즘으로 같은 증상을 만들어낼 수 있습니다. 다음 글에서 이어가겠습니다.

---

## 참고 자료

| 자료 | URL |
|---|---|
| WiredTiger Cache & Eviction Tuning | https://source.wiredtiger.com/mongodb-6.0/tune_cache.html |
| MongoDB `$set` Operator | https://www.mongodb.com/docs/manual/reference/operator/update/set/ |
| Transactions Production Considerations | https://www.mongodb.com/docs/manual/core/transactions-production-consideration/ |
| Replication Source README | https://github.com/mongodb/mongo/blob/master/src/mongo/db/repl/README.md |
| Troubleshoot Replica Sets | https://www.mongodb.com/docs/manual/tutorial/troubleshoot-replica-sets/ |
| Replica Set Data Synchronization | https://www.mongodb.com/docs/manual/core/replica-set-sync/ |
| Replica Set Oplog | https://www.mongodb.com/docs/manual/core/replica-set-oplog/ |
| MongoDB 8.0 Release Notes | https://www.mongodb.com/docs/manual/release-notes/8.0/ |
