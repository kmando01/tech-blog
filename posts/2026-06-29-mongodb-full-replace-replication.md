# MongoDB replaceOne이 secondary를 '멈춘다'는 말은 절반만 맞다

> **검증일**: 2026-06-29 | **대상**: MongoDB 공식 문서 1차 자료 검증 | **시리즈**: 분산 시스템 실증 실험

---

"MongoDB에서 replaceOne을 쓰면 secondary가 45초 멈춘다."

운영 중 실제로 이런 현상을 겪은 팀의 보고가 있었습니다. 그런데 "45초 멈춘다"는 표현이 정확한지, 그 메커니즘이 어디서 오는지를 공식 문서로 확인했더니 5가지 중요한 사실이 추가로 나왔습니다.

결론부터 말하면: secondary가 완전히 "멈추는" 건 아닙니다. 하지만 실제로 일어나는 일은 더 복잡하고, 상황에 따라서는 45초보다 훨씬 심각할 수 있습니다.

---

## 왜 이걸 확인했는가

[replaceOne vs $set 실험](./2026-06-29-mongodb-replace-vs-set.md)에서 oplog 크기가 175배 차이난다는 것을 직접 측정했습니다. 175x oplog가 복제에 어떤 영향을 주는지, 특히 "secondary가 멈춘다"는 현상의 정확한 메커니즘을 공식 문서로 검증했습니다.

5개 항목 모두 MongoDB 공식 문서로 확정됐습니다. 그중 가장 중요한 것은 **oplog 동작 방식**과 **MongoDB 8.0의 변경 사항**입니다.

---

## V1. Secondary는 실제로 "멈추지" 않습니다

"45초 동안 secondary 읽기가 전부 막힌다"는 표현은 정확하지 않습니다.

MongoDB 공식 문서의 설명입니다.

> *"Read operations that target secondaries and are configured with a read concern level of 'local' or 'majority' read from a WiredTiger snapshot of the data if the read takes place on a secondary where replication batches are being applied."*
> — MongoDB Replica Set Data Synchronization

> *"This allows the read to occur simultaneously with replication, while still guaranteeing a consistent view of the data."*
> — MongoDB FAQ: Concurrency

secondary가 replication batch를 적용하는 도중에도 `local`·`majority` read concern을 쓰는 읽기는 WiredTiger snapshot에서 처리됩니다. 읽기가 막히는 게 아니라, replication 직전 시점의 snapshot 데이터를 읽습니다.

즉, "45초 동안 secondary 읽기가 안 된다"가 아니라 "45초 동안 secondary 읽기가 stale 데이터를 반환한다"가 더 정확한 표현입니다.

혹시 secondary read를 쓰고 있는데 "45초 동안 응답이 없었다"는 경험이 있으신가요? 그렇다면 읽기 차단이 아닌 다른 원인일 가능성이 있습니다.

---

## V2. 45초 lag의 원인은 세 가지입니다

"replaceOne이 secondary에 45초 lag을 만든다"는 현상이 실제로 발생했다면, 원인이 세 가지 중 하나입니다.

### 시나리오 A: 실제 replication lag (원래 예상 메커니즘)

replaceOne의 100KB oplog entry가 secondary에서 chained replay되는 시간이 누적되어 45초 lag이 생기는 경우입니다. 이게 원래 예상했던 메커니즘입니다.

### 시나리오 B: 트랜잭션 60초 한계 + abort 재시도 누적

MongoDB 공식 문서입니다.

> *"Transactions have a lifetime limit as specified by `transactionLifetimeLimitSeconds`. The default is 60 seconds."*
> — MongoDB Limits and Thresholds

> *"By default, a transaction must have a runtime of less than one minute. ... Transactions that exceed this limit are considered expired and will be aborted by a periodic cleanup process."*
> — MongoDB Production Considerations

replaceOne을 트랜잭션 안에서 쓰고 있었다면, 100KB 문서 교체가 60초 한계에 걸려 abort될 수 있습니다. abort 후 재시도가 누적되면 클라이언트 관점에서 45초 지연처럼 보일 수 있습니다. 이 경우 실제 replication은 정상이고, 클라이언트 재시도 로직이 문제입니다.

### 시나리오 C: WiredTiger cache 초과 → TransactionTooLargeForCache abort

> *"If a transaction is too large to ever fit in the WiredTiger cache, the transaction aborts and returns a `TransactionTooLargeForCache` error."*
> — MongoDB Production Considerations

트랜잭션이 WiredTiger cache를 초과하면 abort됩니다. replaceOne으로 대량 문서를 한 트랜잭션에 묶으면 이 에러가 발생할 수 있습니다. 이 경우 replication에 도달조차 못하고 abort됩니다.

**어느 시나리오인지 진단하는 방법:**

```bash
# 로그에서 키워드 검색
grep "TransactionTooLargeForCache" mongod.log  # 시나리오 C
grep "abortTransaction" mongod.log             # 시나리오 B (60초 전후 타임스탬프 확인)
# 둘 다 없으면 시나리오 A

# 실험 시 트랜잭션 abort 카운터 측정 추가
db.serverStatus().transactions.totalAborted  # before/after 비교
```

---

## V3. Oplog는 설정 크기를 '넘어서' 성장합니다 (4.0+)

replaceOne의 175x oplog가 secondary lag을 만들면, oplog는 어떻게 됩니까?

```
replaceOne oplog entry: ~100 KB
$set oplog entry:       ~0.6 KB
```

"oplog 크기를 제한해두면 괜찮다"는 생각은 틀렸습니다. MongoDB 4.0부터 oplog는 configured size를 넘어서 성장합니다.

> *"Unlike other capped collections, the oplog can grow past its configured size limit to avoid deleting the majority commit point."*
> — MongoDB Replica Set Oplog

> *"Starting in 4.0, if the replication majority commit point lags past where the oplog would normally truncate, instead the oplog will grow to avoid deleting the majority commit point. This means that the oplog size configuration is now a **minimum size for the oplog, not a maximum**."*
> — MongoDB JIRA DOCS-11484

설정값은 **최솟값**이지 최댓값이 아닙니다. replaceOne으로 secondary lag이 커지면, oplog를 truncate하지 않고 디스크가 계속 늘어납니다.

"Oplog 윈도우 40분에서 72시간으로 회복됐다"는 표현보다 "Oplog 디스크 bloat이 회복됐다"가 더 정확합니다. 윈도우와 디스크 사용량을 모두 기록해야 합니다.

```bash
# 모니터링 시 두 가지 모두 기록 권장
rs.printReplicationInfo()      # timeDiffHours (윈도우)
db.oplog.rs.stats().maxSize    # 현재 oplog 크기 (bloat 감지)
```

---

## V4. MongoDB 8.0에서 writer/applier가 분리됐습니다

MongoDB 8.0부터 secondary가 oplog batch를 처리하는 방식이 바뀌었습니다.

> *"Starting in MongoDB 8.0, secondaries write and apply oplog entries for each batch in parallel. A writer thread reads new entries from the primary and writes them to the local oplog. An applier thread asynchronously applies these changes to the local database. This introduces a breaking change for `metrics.repl.buffer`."*
> — MongoDB Release Notes 8.0

7.0 이하에서는 oplog 수신과 적용이 하나의 buffer에서 처리됐습니다. 8.0부터 두 단계로 분리됩니다.

**측정 지표가 바뀌었습니다:**

| | 7.0 이하 | 8.0+ |
|--|---------|------|
| 수신 buffer | `metrics.repl.buffer.*` | `metrics.repl.buffer.write.*` |
| 적용 buffer | (동일) | `metrics.repl.buffer.apply.*` |

8.0에서 복제 lag을 모니터링할 때는 write buffer가 안정한데 apply buffer(`metrics.repl.buffer.apply.sizeBytes`)가 쌓이면 secondary applier 단계가 병목입니다. 7.0 기준으로 만든 모니터링 대시보드는 8.0 환경에서 잘못된 값을 읽을 수 있습니다.

---

## V5. MongoDB 8.0에서 w:majority가 더 일찍 반환됩니다

> *"Starting in MongoDB 8.0, write operations that use the 'majority' write concern return an acknowledgment when the majority of replica set members have written the oplog entry for the change. ... In previous releases, these operations would wait and return an acknowledgment after the majority of replica set members applied the change."*
> — MongoDB 8.0 Compatibility Changes

| 버전 | w:majority 반환 시점 |
|------|---------------------|
| 7.0 이하 | secondary가 변경 사항을 **적용(applied)** 완료할 때 |
| 8.0+ | secondary가 oplog entry를 **수신(wrote)** 완료할 때 |

8.0에서는 commit이 반환된 후에도 secondary의 apply가 진행 중일 수 있습니다. "commit 직후 lag"을 측정하면 7.0보다 lag이 남아있는 것처럼 보입니다. 두 버전의 측정값을 직접 비교하면 안 됩니다.

replaceOne의 175x oplog가 secondary 적용에 걸리는 시간은 7.0에서 commit에 포함됐지만, 8.0에서는 commit 이후에도 계속 측정됩니다.

---

## 5가지를 종합하면

| 검증 항목 | 결론 | 실무 영향 |
|-----------|------|----------|
| V1. Secondary reads | Snapshot에서 처리됨 → "멈춤" 아님 | "45초 응답 없음"은 다른 원인 |
| V2. 트랜잭션 60초 한계 | abort + 재시도 누적 가능 | 로그에서 `totalAborted` 확인 |
| V3. Oplog 무한 성장 | configured size는 최솟값 | 디스크 bloat 별도 모니터링 |
| V4. writer/applier 분리 | 8.0에서 측정 지표 변경 | 버전별 다른 필드명 사용 |
| V5. w:majority 의미 변경 | 8.0에서 더 일찍 반환 | 버전 간 lag 측정값 비교 불가 |

"replaceOne이 secondary를 45초 멈춘다"는 현상을 재현하거나 모니터링하려면 이 5가지를 먼저 확인해야 합니다. 특히 **실험 버전 고정**이 핵심입니다. 7.0과 8.0은 지표 이름부터 동작 시점까지 달라서, 같은 코드로 같은 환경을 재현해도 수치가 다르게 나옵니다.

---

## 모니터링 체크리스트

```bash
# 1. secondary lag 기본 확인
rs.printSecondaryReplicationInfo()

# 2. oplog bloat 감시 (4.0+)
rs.printReplicationInfo()      # timeDiffHours
db.oplog.rs.stats().maxSize    # 실제 디스크 크기

# 3. 트랜잭션 abort 추적
db.serverStatus().transactions.totalAborted

# 4. 복제 buffer 확인 (버전별 다름)
# MongoDB 7.0 이하:
db.serverStatus().metrics.repl.buffer

# MongoDB 8.0+:
db.serverStatus().metrics.repl.buffer.apply.sizeBytes  # 적용 대기
db.serverStatus().metrics.repl.buffer.write.sizeBytes  # 수신 대기

# 5. write queue 확인
db.serverStatus().globalLock.currentQueue.writers
```

---

## 관련 글

- [MongoDB replaceOne vs $set: 예측이 두 번 빗나갔다](./2026-06-29-mongodb-replace-vs-set.md) — dirty bytes 1.9x, oplog 175x 직접 측정
- [핫 도큐먼트는 자기만 느린 게 아니다](./2026-06-29-mongodb-hot-document.md) — write ticket pool과 collateral damage

---

## 참고 자료

| 공식 문서 | 확인 항목 |
|-----------|----------|
| [MongoDB 8.0 Release Notes](https://www.mongodb.com/docs/manual/release-notes/8.0/) | writer/applier 분리, buffer 지표 변경 |
| [MongoDB 8.0 Compatibility Changes](https://www.mongodb.com/docs/manual/release-notes/8.0-compatibility/) | w:majority 의미 변경 |
| [Limits and Thresholds](https://www.mongodb.com/docs/manual/reference/limits/) | transactionLifetimeLimitSeconds 기본값 |
| [Transactions Production Considerations](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/) | TransactionTooLargeForCache |
| [Replica Set Data Synchronization](https://www.mongodb.com/docs/manual/core/replica-set-sync/) | Secondary snapshot reads |
| [FAQ: Concurrency](https://www.mongodb.com/docs/manual/faq/concurrency/) | Read during replication |
| [Replica Set Oplog](https://www.mongodb.com/docs/manual/core/replica-set-oplog/) | Oplog 무한 성장 |
