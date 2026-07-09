---
title: "Replay 가능한 변경 이력 이벤트를 위한 Claim-and-Check 패턴"
description: "Kafka 이벤트에는 replay에 필요한 claim만 남기고, 재처리나 복구 시 Envers audit log와 transaction context로 변경 전후 값을 다시 재구성하는 Claim-and-Check 패턴을 정리했습니다."
category: "Backend Engineering"
pubDate: "2026-05-27T00:00:00+09:00"
---

## TL;DR

변경 이력 시스템을 만들 때 가장 쉬운 방법은 Kafka 이벤트에 변경 전후 값을 모두 담는 것입니다.

```json
{
  "entityType": "Campaign",
  "entityId": 123,
  "before": { "...": "..." },
  "after": { "...": "..." }
}
```

하지만 이 방식은 빠르게 무거워집니다.

```text
payload size 증가
producer가 entity별 diff 로직을 알아야 함
schema 변경 시 producer/consumer 동시 영향
민감 필드 filtering 책임이 producer로 번짐
```

이번 설계에서는 반대로 접근했습니다.

Kafka에는 변경 전후 값을 싣지 않았습니다.

대신 "무엇이 바뀌었다"는 claim만 발행했습니다.

```json
{
  "entityType": "Campaign",
  "entityId": 123,
  "mutationType": "UPDATE",
  "executionId": "01J...",
  "publishedAt": "2026-05-27T00:00:00Z"
}
```

consumer는 이 claim을 받은 뒤 Envers audit table과 transaction context를 조회해 변경 전후 값을 재구성합니다.

이 방식은 Kafka 메시지를 작게 만들고 producer coupling을 줄입니다.

하지만 공짜는 아닙니다.

consumer가 audit storage를 조회해야 하고, event delivery guarantee는 별도로 설계해야 합니다.

이번 글의 결론은 하나입니다.

> 변경 전후 값을 Kafka에 직접 담지 않아도 변경 이력은 재구성할 수 있다. 다만 그 전제는 audit log가 신뢰 가능한 source of truth로 남아 있고, Kafka 이벤트는 payload가 아니라 lookup claim이라는 책임만 가져야 한다는 점이다.

## 1. 문제 상황

사용자가 어떤 엔티티를 수정했을 때, 변경 이력 화면에는 다음 정보가 필요했습니다.

```text
누가 바꿨는가
언제 바꿨는가
어떤 엔티티가 바뀌었는가
무슨 필드가 바뀌었는가
변경 전 값은 무엇인가
변경 후 값은 무엇인가
```

처음 보면 단순한 audit log 문제처럼 보입니다.

하지만 요구사항을 자세히 보면 두 가지 성격이 섞여 있습니다.

```text
1. 변경 사실을 빠르게 전달해야 한다
2. 변경 전후 값을 정확하게 재구성해야 한다
```

이 둘을 같은 Kafka payload에 모두 넣으면 구조가 무거워집니다.

예를 들어 producer가 변경 전후 값을 모두 계산해서 이벤트에 담는다고 해보겠습니다.

```text
entity update request
↓
load before state
↓
apply change
↓
load after state
↓
calculate diff
↓
publish Kafka event with before/after
```

이 방식은 직관적입니다.

하지만 producer가 너무 많은 책임을 갖게 됩니다.

## 2. before/after payload 방식의 문제

Kafka 이벤트에 before/after를 직접 담으면 consumer는 편합니다.

이벤트 하나만 읽으면 화면에 필요한 값을 거의 바로 만들 수 있습니다.

하지만 비용은 producer와 message model에 쌓입니다.

```text
producer가 entity별 diff 규칙을 알아야 함
field masking / filtering 책임이 producer에 생김
entity schema 변경이 event schema 변경으로 번짐
payload가 entity 크기에 비례해 커짐
nested child가 많으면 message size가 빠르게 증가함
```

특히 변경 이력은 대상 엔티티가 하나가 아닙니다.

```text
Campaign
AdGroup
Ad
Affiliation
ExcludeProduct
...
```

각 엔티티마다 변경 의미가 다릅니다.

같은 필드 변경이라도 UI에 보여줄 문구와 grouping 방식이 다를 수 있습니다.

producer가 모든 엔티티의 before/after rendering 규칙을 알게 되면, 변경 이력 시스템이 도메인 서비스 내부로 깊게 새어 들어옵니다.

또 다른 문제는 메시지 크기입니다.

변경 전후 전체 payload를 담으면 Kafka 이벤트 크기는 entity 구조에 끌려갑니다.

```text
message size
≈
metadata
+
before payload
+
after payload
+
extras
```

child entity가 많거나 nested structure가 깊으면 이벤트가 계속 커집니다.

이전 글에서 다룬 Kafka size 문제와 같은 방향의 위험입니다.

## 3. 선택한 방향: Claim-and-Check

선택한 방향은 Claim-and-Check였습니다.

Kafka에는 변경 전후 값을 직접 넣지 않습니다.

Kafka 이벤트에는 조회에 필요한 claim만 넣습니다.

```kotlin
data class EntityChangeEventClaimMsg(
    val entityType: String,
    val entityId: Long,
    val mutationType: String,
    val executionId: String,
    val publishedAt: Instant,
    val extras: Map<String, String> = emptyMap(),
)
```

이 메시지는 "무슨 일이 있었다"는 신호입니다.

실제 변경 전후 값은 consumer가 별도 저장소에서 조회합니다.

이번 구조에서 그 저장소는 Envers audit table과 transaction context였습니다.

```text
Kafka claim
↓
consumer
↓
lookup Envers audit table by entityType/entityId/revision
↓
lookup transaction context by executionId
↓
reconstruct before/after
↓
store change history read model
```

Kafka 이벤트의 책임은 작아집니다.

```text
entityType
entityId
mutationType
executionId
publishedAt
extras
```

이 정도만 있으면 consumer가 필요한 값을 찾아갈 수 있습니다.

## 4. 왜 Envers를 check 저장소로 썼나

이미 Hibernate Envers가 엔티티 변경 이력을 audit table에 남기고 있었습니다.

Envers는 audited entity의 변경을 별도 audit table에 저장하고, revision 개념으로 변경 시점을 추적할 수 있습니다.

즉 변경 전후 값을 재구성하기 위한 원천 데이터가 이미 존재했습니다.

그렇다면 Kafka에 같은 값을 다시 싣는 것은 중복입니다.

```text
DB audit table
→ 이미 변경 이력의 source of truth

Kafka event
→ 변경 사실을 consumer에게 알리는 trigger
```

이렇게 책임을 나누면 producer는 간단해집니다.

producer는 entity가 바뀌었다는 사실만 발행합니다.

consumer는 audit table을 기준으로 before/after를 계산합니다.

```text
producer responsibility
trigger delivery

consumer responsibility
history reconstruction

audit storage responsibility
historical state source
```

이 분리는 중요합니다.

변경 이력의 정확성은 Kafka payload가 아니라 audit storage의 일관성에 기대게 됩니다.

## 5. event 발행 시점

claim event는 DB transaction commit 이후에 발행해야 합니다.

commit 전에 Kafka 이벤트가 나가면 consumer가 아직 존재하지 않는 revision을 조회할 수 있습니다.

```text
publish Kafka event before commit
↓
consumer receives claim
↓
consumer queries audit table
↓
revision not committed yet
```

또는 더 나쁜 경우도 있습니다.

```text
Kafka publish success
↓
DB rollback
↓
consumer received claim for data that never committed
```

그래서 event 발행은 transaction commit 이후로 미뤘습니다.

```kotlin
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
fun handle(event: EntityChangeDetectedEvent) {
    kafkaProducer.send(EntityChangeEventClaimMsg.from(event))
}
```

이 구조는 rollback된 변경에 대한 claim 발행을 막습니다.

다만 이전 글에서 정리했듯이, `AFTER_COMMIT`은 DB와 Kafka publish의 원자성을 보장하지 않습니다.

여기서는 역할을 분리해서 봐야 합니다.

```text
AFTER_COMMIT
→ committed revision만 claim으로 발행하기 위한 장치

Outbox
→ committed claim intent를 복구 가능하게 남기는 장치
```

이번 설계의 핵심은 Claim-and-Check이고, delivery guarantee를 더 강하게 가져가려면 Outbox를 추가로 검토해야 합니다.

## 6. async를 쓰지 않은 이유

event listener는 비동기로 처리하고 싶어질 수 있습니다.

하지만 request-scoped context가 있으면 주의해야 합니다.

이번 경우에는 변경 이력에 필요한 transaction context가 request scope에 묶여 있었습니다.

```text
request thread
↓
HistoryTransactionInfo available
↓
publish domain event
↓
AFTER_COMMIT listener
```

여기서 `@Async`로 스레드를 바꾸면 request-scoped context에 접근할 수 없습니다.

```text
new async thread
↓
request scoped bean not available
↓
missing executionId / actor context
```

그래서 listener 자체는 동기 흐름으로 두고, Kafka client의 send가 내부적으로 비동기 처리되는 구조를 이용했습니다.

중요한 것은 `@Async`를 쓰지 않았다는 사실이 아니라, 어떤 context가 thread boundary를 넘어갈 수 없는지 명확히 했다는 점입니다.

request context가 필요한 코드는 thread 전환 전에 필요한 값을 message로 고정해야 합니다.

## 7. 재구성 흐름

consumer는 claim을 받으면 변경 이력을 재구성합니다.

단순화하면 다음과 같습니다.

```text
1. claim 수신
2. entityType으로 audit table 결정
3. entityId로 revision 후보 조회
4. executionId 또는 revision metadata로 같은 transaction 변경 묶음 확인
5. 이전 revision과 현재 revision 비교
6. 변경 field 추출
7. UI용 change history read model 저장
```

핵심은 Kafka event가 complete data가 아니라는 점입니다.

Kafka event는 다음 조회를 가능하게 하는 key set입니다.

```text
entityType
entityId
executionId
publishedAt
```

이 key set이 불안정하면 consumer는 재구성할 수 없습니다.

따라서 Claim-and-Check에서 message contract는 작지만 가볍게 보면 안 됩니다.

작은 메시지일수록 각 필드의 의미가 더 중요해집니다.

## 8. whitelist가 필요한 이유

모든 entity 변경을 Kafka로 보내면 안 됩니다.

변경 이력 화면에서 의미 있는 엔티티만 대상이어야 합니다.

```text
Campaign
AdGroup
Ad
Affiliation
ExcludeProduct
...
```

화이트리스트를 두면 producer는 변경 감지 범위를 제어할 수 있습니다.

```kotlin
fun resolveEntityType(entity: Any): EntityType? {
    return when (entity) {
        is Campaign -> EntityType.CAMPAIGN
        is AdGroup -> EntityType.AD_GROUP
        is Ad -> EntityType.AD
        else -> null
    }
}
```

화이트리스트 밖의 엔티티는 skip합니다.

이 방식은 지루하지만 안전합니다.

자동으로 모든 audited entity를 발행하면 노이즈가 많아집니다.

```text
내부 관리 entity
중간 mapping table
derived state
UI에 보여줄 필요 없는 변경
```

변경 이력은 "모든 DB 변경 로그"가 아닙니다.

사용자가 이해할 수 있는 product-level history입니다.

따라서 발행 대상은 명시적으로 좁히는 편이 낫습니다.

## 9. 장점

Claim-and-Check를 선택하면 가장 먼저 메시지가 작아집니다.

```text
before/after payload 없음
entity snapshot 없음
lookup key만 포함
```

producer coupling도 줄어듭니다.

producer는 변경 전후 값을 어떻게 보여줄지 몰라도 됩니다.

```text
producer
→ changed signal

consumer
→ history reconstruction
```

schema 변경에도 비교적 강합니다.

entity field가 늘어나도 Kafka event schema가 매번 커지지 않습니다.

before/after 계산 규칙은 consumer 쪽에서 audit schema를 기준으로 발전시킬 수 있습니다.

민감 필드 처리도 한 곳으로 모을 수 있습니다.

```text
masking
field label mapping
ignored field filtering
grouping
display transformation
```

이런 로직이 producer마다 흩어지지 않습니다.

## 10. 한계

이 구조의 가장 큰 한계는 consumer가 check 저장소에 의존한다는 점입니다.

Kafka event만으로는 변경 이력을 완성할 수 없습니다.

```text
Kafka available
audit table unavailable
↓
history reconstruction delayed
```

audit table schema가 바뀌면 consumer도 영향을 받습니다.

revision timestamp precision도 중요합니다.

초 단위 timestamp만으로 여러 변경을 정렬하면 순서가 흔들릴 수 있습니다.

따라서 deterministic sort key가 필요합니다.

```text
revision number
revision timestamp
entity id
event id
sequence
```

또 다른 한계는 delivery guarantee입니다.

`AFTER_COMMIT` listener에서 Kafka publish가 실패하면 DB에는 audit row가 있는데 claim event가 없을 수 있습니다.

```text
DB commit success
audit row exists
Kafka publish failed
consumer never notified
```

이 문제를 강하게 막으려면 Outbox가 필요합니다.

Claim-and-Check는 payload coupling을 줄이는 패턴이지, delivery guarantee를 자동으로 해결하는 패턴은 아닙니다.

## 11. Outbox와의 관계

Claim-and-Check와 Outbox는 다른 문제를 풉니다.

| 패턴 | 해결하는 문제 | 남는 문제 |
|---|---|---|
| Claim-and-Check | Kafka payload를 작게 만들고 producer coupling을 줄임 | claim 자체가 유실될 수 있음 |
| Outbox | commit된 event intent를 복구 가능하게 남김 | payload 크기와 lookup 설계는 별도 문제 |

둘은 경쟁 관계가 아닙니다.

조합할 수 있습니다.

```text
DB transaction
↓
business change
audit table write
outbox claim row insert
↓
publisher reads outbox
↓
Kafka claim event
↓
consumer reconstructs from audit table
```

이렇게 하면 event intent도 DB에 남고, Kafka 메시지도 작게 유지됩니다.

대신 운영 비용은 늘어납니다.

```text
outbox table
publisher retry
duplicate handling
consumer idempotency
outbox cleanup
```

따라서 선택 기준은 명확해야 합니다.

```text
일부 변경 이력 누락을 재처리로 복구할 수 있는가
→ Claim-and-Check + monitoring으로 시작 가능

변경 이력 누락이 감사/정산/고객 신뢰 문제인가
→ Outbox까지 포함해야 함
```

## 12. 테스트에서 확인할 것

이 설계는 단순히 "Kafka 메시지가 발행됐다"만 테스트하면 부족합니다.

다음 케이스를 봐야 합니다.

```text
1. DB rollback 시 claim event가 발행되지 않는가
2. commit 이후 claim event가 발행되는가
3. claim event에 before/after payload가 들어가지 않는가
4. consumer가 audit table에서 before/after를 재구성하는가
5. request-scoped context 없이 async listener가 동작하지 않는 위험을 알고 있는가
6. whitelist 밖 entity 변경은 skip되는가
7. 같은 timestamp의 여러 revision이 deterministic하게 정렬되는가
8. Kafka publish 실패 시 재처리 경로가 있는가
```

특히 3번은 중요합니다.

시간이 지나면 누군가 편의상 payload를 다시 넣고 싶어집니다.

그 순간 Claim-and-Check의 장점이 사라집니다.

message contract를 작게 유지하는 것도 테스트해야 할 설계 속성입니다.

## 13. 체크리스트

변경 이력 이벤트를 설계한다면 다음 순서로 보겠습니다.

```text
1. Kafka event가 complete payload인지 lookup claim인지 결정했는가
2. before/after의 source of truth가 어디인지 정했는가
3. audit table에서 이전/현재 revision을 안정적으로 찾을 수 있는가
4. transaction commit 이전 event 발행을 막았는가
5. request context가 thread boundary를 넘는지 확인했는가
6. event 대상 entity whitelist가 있는가
7. field masking과 display mapping 책임이 한 곳에 모여 있는가
8. timestamp tie-breaker가 있는가
9. Kafka publish 실패 시 재처리 전략이 있는가
10. Outbox가 필요한 이벤트인지 판단했는가
```

이 체크리스트의 핵심은 하나입니다.

Kafka에 무엇을 넣지 않을지 먼저 정해야 합니다.

## 14. 정리

이번 설계는 Kafka 이벤트를 작게 유지하는 방향이었습니다.

변경 전후 값을 Kafka에 싣지 않고, claim만 발행했습니다.

consumer는 Envers audit table과 transaction context를 조회해 변경 이력을 재구성했습니다.

이 방식의 장점은 분명합니다.

```text
작은 Kafka message
낮은 producer coupling
중앙화된 history reconstruction
schema 변경 영향 축소
```

하지만 한계도 분명합니다.

```text
consumer가 audit storage에 의존
claim event 유실 가능
revision ordering 문제
Outbox 필요 가능성
```

이번 사례의 교훈은 간단합니다.

> Kafka 이벤트는 모든 데이터를 담는 봉투가 아니라, 필요한 데이터를 찾아갈 수 있게 하는 claim일 수 있다. 중요한 것은 payload를 줄이는 것이 아니라 source of truth와 재구성 책임을 명확히 나누는 것이다.

변경 이력 시스템은 이벤트를 많이 보내는 시스템이 아닙니다.

사용자가 이해할 수 있는 변경 사실을 안정적으로 재구성하는 시스템입니다.

## References

- [Hibernate Envers][hibernate-envers].
- [Hibernate ORM User Guide, Envers][hibernate-envers-user-guide].
- [Spring Framework, Transaction-bound Events][spring-transaction-bound-events].
- [Spring Framework Javadoc, TransactionalEventListener][spring-transactional-event-listener].
- [Mary Shaw, ICSE 2003][shaw-icse-2003].

[hibernate-envers]: https://hibernate.org/orm/envers/
[hibernate-envers-user-guide]: https://docs.hibernate.org/orm/current/userguide/html_single/
[spring-transaction-bound-events]: https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html
[spring-transactional-event-listener]: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionalEventListener.html
[shaw-icse-2003]: https://www.cs.cmu.edu/~Compose/shaw-icse03.pdf
