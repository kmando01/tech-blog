# MongoDB secondary read 라우팅이 조용히 깨지는 이유 — @MongoSecondaryRead 로 봉인하기

> **작성일**: 2026-06-29 | **환경**: Spring Boot 3.3.4 + Spring Data MongoDB 4.3.x | 시리즈: MongoDB 실증 실험

read replica를 붙이고 나서 `secondaryPreferred` 설정을 공들여 완성했는데, 모니터링 대시보드는 여전히 master 한 곳만 바쁘다면 — 지금부터 읽어보세요. 저희 팀이 정확히 그 상황이었고, 원인은 아주 조용한 곳에 숨어 있었습니다.

---

## GitHub 이슈로 시작된 질문

이 문제를 파고들다 보면 두 개의 Spring Data MongoDB 이슈를 만나게 됩니다.

**[DATAMONGO-2103 / #2971](https://github.com/spring-projects/spring-data-mongodb/issues/2971)** — 2018년에 열린 이슈입니다. 요약하면 이렇습니다.

> "현재 ReadPreference는 MongoTemplate 레벨에서만 설정 가능합니다. Repository와 Query 메서드 레벨에서 선언할 수 있는 어노테이션이 필요합니다."

이 요청은 결국 Spring Data MongoDB 4.2 (Spring Boot 3.1)에서 `@ReadPreference` 어노테이션으로 구현됐습니다. 그런데 막상 써보면 secondary 라우팅이 여전히 안 되는 경우가 있습니다.

**[#5019](https://github.com/spring-projects/spring-data-mongodb/issues/5019)** — 2024년 이슈입니다. Spring Boot 3.3.x에서 `@Transactional`을 써도 트랜잭션 롤백이 안 된다는 내용인데, Spring 팀 멤버 `mp911de`의 답변이 핵심을 짚습니다.

> "`MongoTemplate`은 반드시 `MongoTransactionManager`에 넘긴 것과 동일한 `MongoDatabaseFactory`로 생성해야 합니다. 그리고 **Spring Boot는 `MongoTransactionManager`를 자동 설정하지 않습니다.**"

이 두 이슈가 오늘 이야기의 배경입니다.

---

## 원인 메커니즘

저희가 겪은 상황은 이렇습니다. `secondaryPreferred` MongoTemplate을 만들어 조회 서비스에 주입했는데, 그 서비스에는 이미 `@Transactional(readOnly = true)`가 붙어있었습니다.

```kotlin
@Service
@Transactional(readOnly = true)  // ← 범인
class EventQueryService(
    @Qualifier("secondaryPreferredTemplate") private val template: MongoTemplate
) {
    fun getEvents(): List<Event> = template.find(...)
}
```

왜 secondary 라우팅이 안 됐을까요? 세 단계를 거칩니다.

**Step 1 — Spring: readOnly는 hint**

Spring `@Transactional#readOnly` Javadoc에 명시된 내용입니다.

> "This just serves as a hint for the actual transaction subsystem; it will not necessarily cause failure of write access attempts. **A transaction manager which cannot interpret the read-only hint will not throw an exception but rather silently ignore the hint.**"

hint입니다. 트랜잭션 매니저가 이해하지 못하면 조용히 무시됩니다.

**Step 2 — MongoTransactionManager: hint 무시 후 트랜잭션 시작**

`MongoTransactionManager.doBegin()` 소스를 열면 `definition.isReadOnly()` 분기가 없습니다. Spring Data MongoDB 공식 문서도 이렇게 명시합니다.

> "`@Transactional(readOnly = true)` advises MongoTransactionManager to **also start a transaction** that adds the ClientSession to outgoing requests."

`readOnly = true`여도 트랜잭션이 시작됩니다.

**Step 3 — MongoDB Driver: 트랜잭션 안 read는 PRIMARY 강제**

MongoDB 공식 매뉴얼입니다.

> "Transactions that contain read operations must use read preference **primary**. All operations in a given transaction must route to the same member."

트랜잭션이 활성화된 순간, `secondaryPreferred` 설정은 드라이버가 조용히 무시합니다. 에러도, 경고도, 로그도 없습니다.

```
@Transactional(readOnly = true)
        │
        ▼
MongoTransactionManager.doBegin()
  └─ readOnly hint 무시 → 트랜잭션 시작
        │
        ▼
MongoDB Java Driver
  └─ 트랜잭션 안 read → ReadPreference 강제로 PRIMARY
        │
        ▼
secondaryPreferredTemplate 설정 무력화 → master 부하 그대로
```

---

## wire 레벨 실험으로 직접 확인

"그럴 것 같다"는 추론에 멈추지 않고 `CommandListener`로 wire 레벨까지 직접 검증했습니다. MongoDB Java Driver는 `CommandStartedEvent`를 통해 서버로 보내는 모든 커맨드를 노출합니다.

트랜잭션 안에서 발행된 커맨드에는 `txnNumber` 필드가 붙습니다. 이 필드 유무로 트랜잭션 활성 여부를 판단했습니다.

```kotlin
class CapturingCommandListener : CommandListener {
    private val events = mutableListOf<CommandStartedEvent>()

    override fun commandStarted(event: CommandStartedEvent) {
        events.add(event)
    }

    fun hasAnyTransaction(): Boolean =
        events.any { it.command.containsKey("txnNumber") }
}
```

**실험 결과 (wire 캡처 직접 출력)**

| 컨텍스트 | wire 출력 | 실제 라우팅 |
|---|---|---|
| `@Transactional(readOnly = true)` | `name=aggregate, txnNumber=true, startTransaction=true` | PRIMARY 강제 ❌ |
| `@Transactional(propagation = NOT_SUPPORTED)` | `name=aggregate, txnNumber=false` | secondaryPreferred 존중 ✅ |

`readOnly = true` 호출 시 `startTransaction=true`와 함께 트랜잭션이 시작되고 `commitTransaction`까지 발행됩니다. 반면 `NOT_SUPPORTED`에서는 `txnNumber` 자체가 없습니다.

그리고 실험 중 한 가지 더 발견했습니다. **Spring Data MongoDB 4.x에서 `MongoTemplate.count()`는 `count` 커맨드가 아니라 `aggregate` 커맨드를 사용합니다.** `$count` stage를 포함한 aggregation pipeline으로 변환됩니다. 커맨드 이름 기반으로 필터하면 아무것도 잡히지 않는 함정이 있습니다.

---

## 해결: @MongoSecondaryRead 커스텀 어노테이션

`Propagation.NOT_SUPPORTED`를 직접 쓰면 의도가 불명확합니다. 왜 NOT_SUPPORTED인지 다음 개발자가 알 수 없습니다. 이 의도를 명시적으로 담은 어노테이션을 만들었습니다.

```kotlin
@Target(AnnotationTarget.FUNCTION, AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@MustBeDocumented
@Transactional(propagation = Propagation.NOT_SUPPORTED)
annotation class MongoSecondaryRead
```

Spring의 `@Transactional`은 메타 어노테이션으로 합성할 수 있어서, 이렇게 선언하면 `@MongoSecondaryRead`가 붙은 메서드에 `NOT_SUPPORTED` propagation이 그대로 적용됩니다.

**클래스 레벨 — 조회 전용 서비스**

```kotlin
@Service
@MongoSecondaryRead
class EventQueryService(
    @Qualifier("secondaryPreferredTemplate") private val template: MongoTemplate
) {
    fun getEvents(): List<Event> = template.find(...)
    fun getEvent(id: String) = template.findById(id, Event::class.java)
}
```

**메서드 레벨 — 읽기/쓰기 혼합 서비스**

```kotlin
@Service
class EventService(
    private val template: MongoTemplate,
    @Qualifier("secondaryPreferredTemplate") private val secondaryTemplate: MongoTemplate
) {
    @MongoSecondaryRead
    fun getEvents(): List<Event> = secondaryTemplate.find(...)   // secondary

    @Transactional
    fun createEvent(event: Event) = template.insert(event)       // primary (트랜잭션)
}
```

write는 별도로 신경 쓰지 않아도 됩니다. MongoDB 드라이버는 ReadPreference에 관계없이 write를 항상 primary로 보냅니다.

---

## Spring Boot 3.4+ 이후는 어떻게 되나

DATAMONGO-2103이 구현된 결과로 Spring Data MongoDB 5.0+ (Spring Boot 3.4+)에서는 Repository 레벨 어노테이션이 가능해졌습니다.

```kotlin
@ReadPreference("secondaryPreferred")
interface EventRepository : MongoRepository<Event, String>
```

이렇게 하면 Template을 별도로 주입하지 않아도 됩니다. 코드가 훨씬 깔끔해집니다.

그런데 `@MongoSecondaryRead`는 3.4+ 이후에도 여전히 필요합니다.

```kotlin
// Spring Boot 3.4+ 조합
@ReadPreference("secondaryPreferred")       // Template 주입 불필요
interface EventRepository : MongoRepository<Event, String>

@Service
@MongoSecondaryRead                          // NOT_SUPPORTED는 여전히 필요
class EventQueryService(private val repo: EventRepository)
```

`@ReadPreference`는 "어디로 보낼지" 선언이고, `@MongoSecondaryRead`는 "트랜잭션을 시작하지 않겠다"는 선언입니다. 두 관심사가 다릅니다. 트랜잭션이 활성화되면 `@ReadPreference`도 무시되기 때문에, 이 둘은 함께 써야 의미가 있습니다.

| Spring Boot 버전 | secondary 선언 | NOT_SUPPORTED 필요 |
|---|---|---|
| 3.3.x | Template 직접 주입 | 필요 |
| 3.4+ | `@ReadPreference` 어노테이션 | **여전히 필요** |

---

## 참고 자료

- [Spring `@Transactional#readOnly` Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html#readOnly()) — "hint, silently ignored"
- [Spring Data MongoDB — Client Session and Transactions](https://docs.spring.io/spring-data/mongodb/reference/mongodb/client-session-transactions.html) — "readOnly=true도 트랜잭션을 시작한다"
- [MongoDB Read Preference 매뉴얼](https://www.mongodb.com/docs/manual/core/read-preference/#std-label-replica-set-read-preference-transactions) — "transactions must use read preference primary" (3곳에서 반복)
- [DATAMONGO-2103 #2971](https://github.com/spring-projects/spring-data-mongodb/issues/2971) — Repository 레벨 `@ReadPreference` 어노테이션 요청 → Spring Data MongoDB 4.2에서 구현
- [spring-data-mongodb #5019](https://github.com/spring-projects/spring-data-mongodb/issues/5019) — `MongoTransactionManager` 수동 등록 필요성 문서화 이슈
