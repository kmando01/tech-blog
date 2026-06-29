# MongoDB replaceOne vs $set: 예측이 두 번 빗나갔다

> **실험일**: 2026-06-29 | **스택**: MongoDB 7.0 / Python pymongo | **시리즈**: 분산 시스템 실증 실험

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

원래 예측(33배)은 "100KB vs 3KB"를 단순 비율로 계산한 것이었는데, oplog entry에는 메타데이터·연산자·필드명이 추가됩니다. `$set`의 실제 oplog 크기(575 bytes)는 순수 데이터(3개 필드) 외에 oplog 구조 overhead가 포함됩니다. 반면 Full Replace의 oplog(100,992 bytes)는 100KB 문서 + overhead로 더 커집니다.

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

### 모니터링 알람 기준

```
cache: tracked dirty bytes / maximum bytes configured > 80%  → warning
globalLock.currentQueue.writers > 20                          → alert
cache: pages evicted by application threads delta > 1/sec     → critical
```

---

## 두 예측이 다른 방향으로 빗나간 이유

같은 "100KB vs 3KB"에서 출발한 예측이 반대 방향으로 틀렸습니다.

- **dirty bytes**: page-level tracking → 두 방식이 같은 page를 dirty → 실제보다 작게 예측
- **oplog**: document-level serialization → full document 직렬화 → 실제보다 크게 예측

WiredTiger의 내부 동작을 두 레이어로 분리해서 이해해야 합니다. 메모리(dirty tracking)는 page 단위, 복제(oplog)는 document 단위입니다. 같은 연산이 레이어마다 다른 비용 구조를 가집니다.

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

실험 코드: [kmando01/mongodb-write-cliff-bench](https://github.com/kmando01/mongodb-write-cliff-bench)
