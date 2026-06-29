# `@Transactional(readOnly = true)`가 MongoDB read replica 부하 분산을 조용히 망가뜨린다

> **실험일**: 2026-06-29 | **환경**: Spring Boot 3.3.4 + MongoDB 7.0 | 시리즈: MongoDB 실증 실험

read replica를 붙였는데 master CPU가 전혀 줄지 않는다면, 코드 어딘가에 `@Transactional(readOnly = true)`가 있는지 먼저 확인해보세요. 저희 팀이 그 함정에 빠졌고, 원인을 규명하는 데 꽤 오랜 시간이 걸렸습니다.

---

## 배경: 부하 분산이 작동하지 않는다

저희 팀은 조회 트래픽을 replica로 분산하기 위해 `secondaryPreferred` 설정을 적용한 별도 `MongoTemplate` 빈을 만들었습니다.

```kotlin
@Configuration
class MongoConfig {
    @Bean("secondaryPreferredTemplate")
    fun secondaryPreferredTemplate(
        mongoDbFactory: MongoDatabaseFactory,
        converter: MappingMongoConverter
    ): MongoTemplate {
        val template = MongoTemplate(mongoDbFactory, converter)
        template.setReadPreference(ReadPreference.secondaryPreferred())
        return template
    }
}
```

설정 자체는 문제가 없어 보였습니다. 그런데 부하 테스트를 돌려보니 master CPU가 전혀 줄지 않았습니다. replica로 쿼리가 단 하나도 가지 않는 상황이었습니다.

원인을 추적하다 보니 조회 서비스 클래스에 이런 코드가 있었습니다.

```kotlin
@Service
@Transactional(readOnly = true)  // ← 이게 범인
class EventReadService(
    @Qualifier("secondaryPreferredTemplate") private val template: MongoTemplate
) {
    fun getEvents(): List<Event> = template.find(...)
}
```

`readOnly = true`는 JPA 환경에서 성능 최적화 힌트로 자주 쓰입니다. 그래서 MongoDB 서비스에도 별생각 없이 붙여뒀던 거였는데, 이게 조용히 모든 read를 primary로 보내고 있었습니다.

---

## 왜 그렇게 되는가

MongoDB Java Driver의 스펙을 먼저 살펴볼 필요가 있습니다.

**트랜잭션 안에서의 read는 ReadPreference를 무시합니다.**

MongoDB 공식 문서에는 이렇게 명시돼 있습니다:

> "Operations in a transaction use the transaction-level read preference. Within a transaction, all data operations must route to the same member."

트랜잭션이 활성화된 상태에서는 어떤 `ReadPreference`를 설정해도 드라이버가 강제로 `PRIMARY`를 사용합니다. `secondaryPreferred`는 silent하게 무시됩니다. 경고도, 로그도 없습니다.

그렇다면 `@Transactional(readOnly = true)`가 실제로 MongoDB 트랜잭션을 시작하는가? 이게 핵심 질문이었습니다.

---

## 실험 설계

"그럴 것 같다"는 추측으로 끝내지 않기로 했습니다. 두 단계로 나눠 wire 레벨까지 직접 검증했습니다.

---

## Stage 1: 트랜잭션이 정말 활성화되는가

**환경**: Testcontainers로 단일 노드 Replica Set을 띄운 JUnit 통합 테스트

MongoDB 트랜잭션은 Replica Set 또는 Sharded Cluster에서만 동작합니다. 단일 standalone 노드에서는 트랜잭션 자체가 지원되지 않으므로, Testcontainers로 RS를 구성했습니다.

**검증 가설**:

| 가설 | 내용 |
|------|------|
| H1 | `@Transactional(readOnly = true)` 메서드 안에서 `TransactionSynchronizationManager.isActualTransactionActive()` = `true` |
| H2 | `readOnly = true` 트랜잭션 안에서 `insert()`가 예외 없이 성공한다 |
| H3 | `readOnly = true` 트랜잭션이 커밋까지 완료된다 (트랜잭션 밖에서 데이터 조회 가능) |

### 첫 번째 시도: H1 실패

첫 실행에서 H1이 실패했습니다.

```
expected: true
 but was: false
```

`@Transactional(readOnly = true)`를 붙였는데도 트랜잭션이 활성화되지 않는다? 처음엔 테스트 설정 문제라고 생각했습니다. 그런데 Spring Boot 공식 문서를 다시 읽어보니 이런 문장이 있었습니다.

> "Spring Boot does not auto-configure a MongoTransactionManager."

**Spring Boot는 `MongoTransactionManager`를 자동으로 등록하지 않습니다.** JPA의 `JpaTransactionManager`와 달리, MongoDB는 수동 등록이 필요합니다. 이 사실을 놓쳤던 겁니다.

```kotlin
@Configuration
class MongoTransactionConfig {
    @Bean
    fun mongoTransactionManager(dbFactory: MongoDatabaseFactory): MongoTransactionManager {
        return MongoTransactionManager(dbFactory)
    }
}
```

이 빈을 추가하자 H1이 통과됐습니다.

### Stage 1 최종 결과

| 가설 | 결과 | 비고 |
|------|------|------|
| H1 | 증명됨 | `MongoTransactionManager` 수동 등록 후 |
| H2 | 증명됨 | `readOnly=true`여도 insert 성공, 예외 없음 |
| H3 | 증명됨 | 커밋 후 트랜잭션 밖에서 데이터 정상 조회 |

H2와 H3가 흥미롭습니다. `readOnly = true`는 Spring에서 트랜잭션 매니저에게 전달하는 힌트일 뿐입니다. `MongoTransactionManager`는 이 힌트에 대해 아무런 제약을 걸지 않습니다. 트랜잭션이 시작되고, write도 허용되고, 커밋까지 됩니다.

---

## Stage 2: wire 레벨에서 직접 확인

Stage 1에서 트랜잭션이 활성화된다는 것을 확인했지만, 실제로 MongoDB 드라이버가 wire 레벨에서 어떤 커맨드를 보내는지를 봐야 했습니다. `ReadPreference`가 정말 무시되는지 코드 수준이 아니라 프로토콜 수준에서 증명하고 싶었습니다.

`CommandListener`를 활용했습니다. MongoDB Java Driver는 모든 커맨드 이벤트를 콜백으로 노출합니다.

```kotlin
class CapturingCommandListener : CommandListener {
    private val events = mutableListOf<CommandStartedEvent>()

    override fun commandStarted(event: CommandStartedEvent) {
        events.add(event)
    }

    fun hasAnyTransaction(): Boolean =
        events.any { it.command.containsKey("txnNumber") }

    fun getEvents(): List<CommandStartedEvent> = events.toList()
}
```

이 리스너를 `MongoClient`에 등록하면 애플리케이션이 MongoDB 서버로 보내는 모든 커맨드를 가로챌 수 있습니다.

**검증 시나리오**:

| 케이스 | 설정 | 예상 |
|--------|------|------|
| B-1 | `@Transactional(propagation = NOT_SUPPORTED)` | 트랜잭션 없음 |
| B-2 | `@Transactional(readOnly = true)` | 트랜잭션 있음 |

### 실제 wire 캡처 출력

**B-2 (`readOnly = true`)**:

```
name=aggregate, txnNumber=true, startTransaction=true   ← 트랜잭션 시작
name=commitTransaction, txnNumber=true                   ← 커밋
```

**B-1 (`NOT_SUPPORTED`)**:

```
name=aggregate, txnNumber=false                          ← 트랜잭션 없음
```

wire 레벨에서 트랜잭션 유무가 명확히 갈립니다. `readOnly = true`는 MongoDB 드라이버 관점에서 완전히 일반 트랜잭션과 동일하게 처리됩니다.

### 예상치 못한 장애물: `count` vs `aggregate`

Stage 2 테스트를 작성하면서 한 번 더 실패를 경험했습니다. 처음에 커맨드 이름으로 `count`를 필터해서 트랜잭션 여부를 확인하려 했는데 커맨드가 잡히지 않았습니다.

원인을 파보니 **Spring Data MongoDB 4.x에서 `MongoTemplate.count()`는 내부적으로 `count` 커맨드가 아니라 `aggregate` 커맨드를 사용합니다.** `$count` stage를 포함한 aggregation pipeline으로 변환되는 겁니다.

```
// Spring Data MongoDB 4.x의 count() 실제 wire 커맨드
{
  "aggregate": "events",
  "pipeline": [{"$count": "count"}],
  ...
}
```

커맨드 이름 기반 필터를 `aggregate`로 바꾸자 테스트가 통과됐습니다. 알려진 변경 사항이지만 실제로 부딪히기 전까지는 체감이 안 됩니다.

---

## 전체 메커니즘 정리

```
@Transactional(readOnly = true)
        │
        ▼
Spring: TransactionDefinition에 readOnly=true 힌트 전달
        │
        ▼
MongoTransactionManager.doBegin()
  ├─ definition.isReadOnly() 분기 없음 (힌트 무시)
  └─ ClientSession 생성 + startTransaction() 호출
        │
        ▼
MongoDB Java Driver
  ├─ 트랜잭션 안의 read → ReadPreference 강제로 PRIMARY
  └─ secondaryPreferred 설정 → 조용히 PRIMARY로 덮어씀
        │
        ▼
PRIMARY 노드로 모든 read 요청 집중 → master 부하 그대로
```

`readOnly = true`는 Spring이 트랜잭션 매니저에게 전달하는 힌트입니다. JPA의 `HibernateJpaDialect`는 이 힌트를 받아 flush mode를 `MANUAL`로 설정하는 등 실질적인 최적화를 적용합니다. 그런데 `MongoTransactionManager`는 이 힌트에 반응하지 않습니다. 힌트가 들어오든 아니든 트랜잭션을 시작합니다.

그리고 트랜잭션이 시작되는 순간, MongoDB 드라이버는 read 요청에 대해 ReadPreference를 강제로 PRIMARY로 고정합니다. 이는 MongoDB의 트랜잭션 일관성 보장을 위한 스펙입니다. 경고 없이, 로그 없이, 조용히 일어납니다.

---

## 해결책

해결책은 단순합니다. 트랜잭션이 필요 없는 조회 서비스에는 트랜잭션을 걸지 않으면 됩니다.

```kotlin
@Service
@Transactional(propagation = Propagation.NOT_SUPPORTED)
class EventReadService(
    @Qualifier("secondaryPreferredTemplate") private val template: MongoTemplate
) {
    fun getEvents(): List<Event> = template.find(...)
}
```

`NOT_SUPPORTED`는 현재 트랜잭션이 있더라도 트랜잭션 없이 실행되도록 강제합니다. 이렇게 하면 MongoDB 드라이버가 ReadPreference를 존중하고 `secondaryPreferred` 설정대로 replica로 요청이 분산됩니다.

버린 선택지도 있었습니다. 클래스 레벨 `@Transactional`을 제거하고 write 메서드에만 다시 붙이는 방법도 고려했는데, 서비스 클래스가 이미 여러 곳에 퍼져있는 상황에서 누락 위험이 있었습니다. `NOT_SUPPORTED`를 클래스 레벨에 선언하면 "이 서비스는 트랜잭션 없이 동작한다"는 의도가 명시적으로 드러나므로 이쪽을 선택했습니다.

---

## 정리: 무엇을 배웠는가

이번 실험을 통해 확인한 내용을 세 가지로 정리하면 다음과 같습니다.

**1. `@Transactional(readOnly = true)`는 MongoDB에서 최적화 힌트가 아니다**

JPA에서 익숙해진 패턴을 MongoDB에 그대로 적용하면 안 됩니다. `MongoTransactionManager`는 `readOnly` 힌트를 무시하고 동일하게 트랜잭션을 시작합니다.

**2. 트랜잭션 안에서 ReadPreference는 무시된다**

이건 버그가 아니라 MongoDB 스펙입니다. 트랜잭션 일관성을 위해 드라이버 레벨에서 의도적으로 PRIMARY를 강제합니다. `secondaryPreferred`를 아무리 잘 설정해도 트랜잭션이 활성화돼 있으면 의미가 없습니다.

**3. `MongoTransactionManager`는 자동 설정되지 않는다**

Spring Boot의 auto-configuration 목록에 `MongoTransactionManager`가 없습니다. `@Transactional`을 쓰려면 반드시 수동으로 빈을 등록해야 합니다. 없으면 `@Transactional`이 붙어있어도 트랜잭션이 시작되지 않으므로, 이미 MongoDB를 쓰는 코드에 `@Transactional`이 있다면 실제로 동작하는지 확인해볼 필요가 있습니다.

---

실험을 설계하고 돌리면서 느낀 건, "당연히 그렇겠지"라는 추측이 가장 위험하다는 점입니다. JPA에서 10년 쓴 패턴이 MongoDB에서는 전혀 다르게 동작할 수 있습니다. 이 글이 같은 함정에 빠진 분들께 도움이 됐으면 합니다.
