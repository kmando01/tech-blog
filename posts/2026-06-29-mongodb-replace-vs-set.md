# MongoDB replaceOne vs $set: 예측이 두 번 빗나갔다

> **실험일**: 2026-06-29 | **스택**: MongoDB 7.0.37 / Python pymongo 4.14 | **시리즈**: 분산 시스템 실증 실험

---

실험을 시작하기 전에 두 가지를 예측했습니다.

첫 번째: `replaceOne`은 100KB 문서를 통째로 쓰니까 WiredTiger dirty bytes가 `$set` 대비 33배 쌓일 것이다.

두 번째: oplog 크기 차이도 33배쯤 날 것이다. 100KB vs 3KB이니까.

결과는 둘 다 틀렸습니다. dirty bytes 비율은 예측의 1/17인 **1.9배**였습니다. oplog 비율은 예측의 5배인 **175배**였습니다. 같은 원리에서 나온 예측이 정반대 방향으로 빗나갔습니다.

---

## 배경: WiredTiger cache와 dirty bytes

MongoDB는 내부 스토리지 엔진으로 WiredTiger를 씁니다. 쓰기 작업은 디스크가 아닌 WiredTiger cache(메모리)에 먼저 반영됩니다. cache에서 아직 디스크로 내려가지 않은 데이터를 dirty bytes라고 합니다.

dirty bytes가 일정 비율(기본 20%)을 넘으면 background eviction worker가 page를 디스크로 flush합니다. 이 비율을 초과하면 application thread가 직접 eviction에 참여하고, write ticket을 소비하면서 다른 쓰기가 대기 줄에 쌓입니다.

`replaceOne`은 100KB 문서 전체를 교체합니다. `$set`은 필드 3개만 바꿉니다. 두 방식이 dirty bytes에 얼마나 다른 압력을 주는지 격리 환경에서 측정했습니다.

---

## 실험 설계

의도적으로 가혹한 조건을 만들었습니다.

```
MongoDB:              7.0.37 (단일 노드 replica set rs0)
WiredTiger cache:     262 MB  (--wiredTigerCacheSizeGB 0.256)
Eviction worker 수:   최소 (threads_min=1, threads_max=2)
컬렉션 크기:          1,000 docs × ~100 KB ≈ 97.8 MB
부하:                 workers 4→16→64→128→256 × 각 60초
```

cache를 작게, eviction worker를 최소화한 이유는 dirty pressure cliff를 빠르게 재현하기 위해서입니다. 차이는 하나뿐입니다.

```python
# Full Replace — 100KB 문서 전체 교체
coll.replace_one({"_id": uid}, full_100kb_doc)

# $set partial — 필드 3개만 업데이트
coll.update_one({"_id": uid}, {"$set": {
    "last_login": now,
    "view_count": cnt,
    "counter": val,
}})
```

> **버전 주의**: 이 실험은 MongoDB 7.0 기준입니다. 8.0에서는 동작이 달라지는 항목이 두 가지 있습니다. 실험 결과 이후에 정리했습니다.

---

## 실측 결과

### Throughput (ops/s)

| workers | Full Replace | $set partial | $set 이점 |
|---------|------------|------------|---------|
| 4 | 8,407 | 12,664 | **+51%** |
| 16 | 9,298 | 11,423 | +23% |
| 64 | 8,191 | 11,548 | +41% |
| 128 | 8,639 | 10,562 | +22% |
| 256 | 6,832 | 8,897 | **+30%** |

$set이 전 구간에서 22~51% 빠릅니다. workers=256에서 Full Replace는 workers=16 대비 **26% 하락**하지만 $set은 **22% 하락**에 그칩니다. 부하가 높을수록 격차가 벌어집니다.

### WiredTiger Dirty Bytes Peak (262 MB 기준 %)

| workers | Full Replace | $set partial |
|---------|------------|------------|
| 4 | **333% [OVERFLOW]** | 102% |
| 16 | **315% [OVERFLOW]** | 100% |
| 64 | **360% [OVERFLOW]** | 98% |
| 128 | **335% [OVERFLOW]** | 101% |
| 256 | **355% [OVERFLOW]** | 101% |

Full Replace는 **모든 부하 단계에서** 설정 cache의 3~3.6배를 초과합니다. $set은 262 MB 경계선 수준에서 안정적으로 유지됩니다.

### Write Queue Peak (write ticket 경합 지표)

| workers | Full Replace | $set partial |
|---------|------------|------------|
| 4 | 0 | 0 |
| 16 | 5 | 0 |
| 64 | **53** | 0 |
| 128 | 0 | 66 |
| 256 | **173** | 0 |

Full Replace에서 workers=64부터 write queue가 폭발합니다. workers=256에서 173. $set은 전 구간 0을 유지합니다 (workers=128의 66은 측정 노이즈로 분류).

---

## 예측 1이 빗나간 이유: dirty bytes 33x → 1.9x

예측은 "100KB vs 3KB = 33배 차이"였습니다. 실측은 **1.9배**였습니다.

이유는 WiredTiger가 dirty 상태를 **page 단위**로 추적하기 때문입니다.

```
WiredTiger dirty tracking:

Full Replace:  100KB page dirty
$set partial:  100KB page dirty (같은 page!)

두 방식 모두 동일한 100KB page를 dirty로 만든다.
page cost = 동일

추가 비용:
  Full Replace: 100KB update chain
  $set:         ~3KB delta

총 비용 비율 = (100KB + 100KB) / (100KB + 3KB) ≈ 1.9×
```

100KB 문서를 수정하면 어떤 방법으로 수정하든 그 page(100KB)가 dirty가 됩니다. `$set`으로 3개 필드만 바꿔도 dirty page 비용은 100KB입니다. 따라서 비율은 33배가 아닌 1.9배입니다.

그런데도 Full Replace의 throughput이 22~51% 낮은 이유가 있습니다. 절대 수치가 다릅니다. Full Replace dirty bytes 평균 442 MB(262MB 기준 169%)는 cache를 넘어 **application thread eviction**을 유발합니다. 이 eviction이 write ticket을 소비하고 write queue를 키웁니다. 비율은 1.9배지만 Full Replace는 항상 한계선을 넘어, $set은 항상 한계선 아래에 머무릅니다.

혹시 "dirty bytes 비율이 1.9배면 큰 차이 아닌 것 아닌가?"라고 생각하셨나요? 수치가 작아 보여도, Full Replace가 항상 300%를 초과하고 $set이 항상 100%에 머문다면 WiredTiger가 경험하는 스트레스는 완전히 다릅니다.

---

## 예측 2가 더 크게 빗나간 이유: oplog 33x → 175x

oplog는 MongoDB가 복제를 위해 기록하는 변경 로그입니다. secondary 노드가 oplog를 읽어 데이터를 동기화합니다. oplog 크기는 secondary lag, 복제 대역폭, 디스크 write throughput 모두에 직접 영향을 줍니다.

```
Full Replace oplog entry: 100,992 bytes (98.6 KB)
$set oplog entry:             575 bytes  (0.6 KB)
비율: 175x   (예측: 33x)
```

예측이 5배 이상 빗나간 이유가 있습니다. Full Replace는 oplog에 **문서 전체를 JSON으로 직렬화**해서 씁니다. 100KB 문서가 그대로 oplog entry에 들어갑니다. `$set`은 변경된 필드 3개의 diff만 씁니다.

### oplog는 설정 크기를 초과해서 성장합니다 (MongoDB 4.0+)

혹시 "oplog 크기를 제한하면 디스크 걱정 없다"고 생각하고 계신가요? MongoDB 4.0부터 oplog는 configured size를 넘어서 성장할 수 있습니다.

공식 문서의 설명입니다.

> *"Unlike other capped collections, the oplog can grow past its configured size limit to avoid deleting the majority commit point."*
> — MongoDB Replica Set Oplog

majority commit point가 뒤처지면 oplog를 truncate하지 않고 디스크를 더 씁니다. Full Replace의 175x oplog가 복제 lag을 유발하면, oplog는 configured size를 넘어 bloat됩니다. "Oplog 윈도우가 줄어든다"보다 "Oplog 디스크가 폭증한다"가 더 정확한 표현입니다.

---

## WiredTiger cache가 동적으로 확장됐습니다

실험 중 흥미로운 현상이 있었습니다. 설정값은 262 MB였는데, 실험 중 `maximum bytes configured` 필드가 **8.5 GB**(Docker VM RAM 한도)로 자동 확장됐습니다.

Full Replace의 dirty bytes(평균 442 MB)가 설정 cache(262 MB)를 초과하자 WiredTiger가 OS 메모리 한도까지 cache를 늘린 것입니다. 이 동적 확장이 없었다면 OOM이나 더 극단적인 성능 저하가 발생했을 것입니다. $set의 dirty bytes(평균 238 MB)는 설정 내에서 안정적으로 운용됐습니다.

---

## 실무 시사점

```python
# 나쁜 예: ODM save() 주의 — 일부 구현이 full replace를 씀
coll.replace_one({"_id": uid}, user_doc)

# 좋은 예: 변경된 필드만 지정
coll.update_one({"_id": uid}, {"$set": {
    "last_login": now,
    "view_count": cnt,
    "counter": val,
}})
```

ORM/ODM을 쓰고 있다면 내부 구현을 확인하세요. **Spring Data MongoDB의 `save()`는 `replaceOne`, `updateFirst()` + `Update`는 partial `$set`**입니다. Mongoose의 `save()`는 기본적으로 dirty field만 전송하지만 `overwrite: true` 옵션 시 full replace가 됩니다.

지금 운영 중인 코드가 replaceOne을 쓰는지 확인하는 가장 빠른 방법은 oplog를 한 번 들여다보는 것입니다. 100KB짜리 oplog entry가 보인다면 Full Replace가 쓰이고 있다는 신호입니다.

### 모니터링 알람 기준

```
cache: tracked dirty bytes / maximum bytes configured > 80%  → warning
globalLock.currentQueue.writers > 20                          → alert
cache: pages evicted by application threads delta > 1/sec     → critical
```

---

## MongoDB 8.0에서 달라지는 두 가지

이 실험은 MongoDB 7.0 기준입니다. 8.0에서 두 가지 동작이 바뀌어서 재현 시 주의가 필요합니다.

### 1. writer/applier thread 분리 → 측정 지표가 바뀌었습니다

8.0부터 secondary가 oplog batch를 writer thread(수신)와 applier thread(적용)로 나눠서 병렬 처리합니다. 이에 따라 복제 모니터링 지표가 구식/신식으로 분리됐습니다.

| | 7.0 이하 | 8.0+ |
|--|---------|------|
| 복제 버퍼 지표 | `metrics.repl.buffer.*` (단일) | `metrics.repl.buffer.apply.*` + `metrics.repl.buffer.write.*` (분리) |

8.0 환경에서 복제 lag을 모니터링할 때는 `metrics.repl.buffer.apply.sizeBytes`를 확인하세요. write buffer는 안정한데 apply buffer가 쌓이면 secondary applier 단계가 병목입니다.

### 2. w:majority가 더 일찍 반환됩니다

7.0에서 `w:"majority"` write concern은 secondary가 변경 사항을 **적용(applied)**할 때 반환했습니다. 8.0부터는 secondary가 oplog entry를 **수신(wrote)**할 때 반환합니다.

```
7.0: commit 반환 = secondary가 변경 적용 완료
8.0: commit 반환 = secondary가 oplog entry 수신 완료 (적용은 진행 중일 수 있음)
```

8.0 환경에서 commit 직후 secondary lag을 측정하면 7.0보다 lag이 남아있을 수 있습니다. 두 버전의 결과를 직접 비교하면 안 됩니다.

---

## 두 예측이 다른 방향으로 빗나간 이유

같은 "100KB vs 3KB"에서 출발한 예측이 반대 방향으로 틀렸습니다.

- **dirty bytes**: page-level tracking → 두 방식이 같은 page를 dirty → 실제보다 작게 예측
- **oplog**: document-level serialization → full document 직렬화 → 실제보다 크게 예측

WiredTiger의 내부 동작을 두 레이어로 분리해서 이해해야 합니다. 메모리(dirty tracking)는 page 단위, 복제(oplog)는 document 단위입니다. 같은 연산이 레이어마다 다른 비용 구조를 가집니다.

replaceOne을 쓰는 코드가 있다면 당장 바꾸는 것보다 먼저 oplog 크기와 dirty bytes 압력을 측정해보는 게 우선입니다. 어떤 문서를 얼마나 자주 교체하는지에 따라 영향 범위가 완전히 달라집니다.

---

## 재현 방법

```bash
docker compose up -d
docker exec cliff-mongo mongosh --eval \
  'rs.initiate({_id:"rs0",members:[{_id:0,host:"localhost:27017"}]})'
sleep 5
pip install pymongo
python3 seed.py
bash run_ladder.sh
python3 analyze.py
```

> **재현 시 버전 고정 권장**: MongoDB 7.0과 8.0은 복제 동작과 측정 지표가 다릅니다. 원 실험과 비교하려면 `image: mongo:7.0`으로 고정하세요.

실험 코드: [kmando01/mongodb-write-cliff-bench](https://github.com/kmando01/mongodb-write-cliff-bench)
