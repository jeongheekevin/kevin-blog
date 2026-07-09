---
title: "TransactionalEventListener만으로 Kafka 발행을 안전하게 만들 수 있을까"
description: "Spring TransactionalEventListener로 DB commit 이후 Kafka 이벤트를 발행할 때 남는 dual-write 위험과 Outbox, Claim-and-Check 전환 기준을 정리했습니다."
category: "Backend Engineering"
pubDate: "2026-05-27T00:00:00+09:00"
---

## TL;DR

Spring 애플리케이션에서 DB 상태를 변경한 뒤 Kafka 이벤트를 발행해야 했습니다.

처음에는 단순한 구조였습니다.

```text
@Transactional service method
↓
DB update
↓
publish application event
↓
@TransactionalEventListener(AFTER_COMMIT)
↓
Kafka publish
```

이 구조는 좋아 보입니다.

DB transaction이 rollback되면 Kafka 이벤트가 나가지 않습니다.

DB commit 이후에만 listener가 실행되므로, consumer가 아직 commit되지 않은 상태를 읽는 문제도 줄어듭니다.

하지만 이것만으로 "DB update와 Kafka publish가 원자적으로 묶였다"고 말하면 안 됩니다.

문제는 commit 이후에 생깁니다.

```text
DB commit success
↓
process crash / Kafka timeout / network error
↓
Kafka publish failed
↓
DB에는 변경이 있지만 event는 없음
```

`@TransactionalEventListener`는 transaction phase에 listener 실행 시점을 묶어주는 도구입니다.

DB와 Kafka 사이의 dual-write 문제를 없애는 도구는 아닙니다.

이번 글의 결론은 하나입니다.

> `@TransactionalEventListener(AFTER_COMMIT)`는 rollback 이벤트 발행을 막는 데 유용하다. 하지만 commit 이후 Kafka 발행 실패까지 복구하려면 Outbox가 필요하고, payload가 계속 커지는 구조라면 Claim-and-Check까지 같이 검토해야 한다.

## 1. 문제 상황

도메인 상태가 변경되면 Kafka로 변경 이벤트를 발행하는 서비스가 있었습니다.

단순화하면 이런 흐름입니다.

```text
update parent entity
update child entities
publish changed event
```

이 이벤트는 downstream system이 상태를 갱신하는 트리거였습니다.

```text
source DB
↓
Kafka event
↓
consumer
↓
cache / search index / derived table refresh
```

따라서 이벤트 발행은 부가 기능이 아니었습니다.

DB 변경이 성공했는데 Kafka 이벤트가 누락되면 downstream 상태가 stale해집니다.

반대로 DB transaction이 rollback됐는데 Kafka 이벤트가 먼저 나가면 존재하지 않는 변경을 consumer가 처리할 수 있습니다.

결국 확인해야 할 것은 두 가지였습니다.

```text
1. rollback된 변경에 대한 event가 나가지 않는가
2. commit된 변경에 대한 event가 반드시 나가는가
```

`@TransactionalEventListener`는 첫 번째 문제에는 꽤 좋은 도구입니다.

하지만 두 번째 문제까지 해결하지는 않습니다.

## 2. naive publish의 문제

가장 단순한 코드는 service method 안에서 바로 Kafka를 호출하는 방식입니다.

```java
@Transactional
public void changeParent(ChangeCommand command) {
    Parent parent = parentRepository.getById(command.parentId());

    parent.change(command);
    childService.replaceChildren(parent, command.children());

    kafkaTemplate.send("parent.changed", ParentChangedEvent.from(parent));
}
```

이 코드는 읽기 쉽습니다.

하지만 transaction 관점에서는 위험합니다.

Kafka send가 DB commit보다 먼저 실행될 수 있기 때문입니다.

```text
Kafka publish success
↓
DB commit failed
↓
consumer sees event for data that was never committed
```

물론 `KafkaTemplate.send()`가 비동기이고 실제 broker ack timing이 다를 수 있습니다.

그렇다고 안전해지는 것은 아닙니다.

핵심은 application code가 DB commit boundary와 Kafka publish boundary를 명확히 분리하지 않았다는 점입니다.

그래서 다음 단계로 `@TransactionalEventListener`를 검토했습니다.

## 3. TransactionalEventListener로 얻는 것

Spring의 `@TransactionalEventListener`는 application event listener를 transaction phase에 맞춰 실행할 수 있게 해줍니다.

기본 phase는 `AFTER_COMMIT`입니다.

명시하면 다음과 같습니다.

```java
@Component
class ParentChangedEventPublisher {

    private final KafkaTemplate<String, ParentChangedEvent> kafkaTemplate;

    ParentChangedEventPublisher(KafkaTemplate<String, ParentChangedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void publish(ParentChangedEvent event) {
        kafkaTemplate.send("parent.changed", event);
    }
}
```

service method는 Kafka를 직접 호출하지 않고 domain event만 발행합니다.

```java
@Transactional
public void changeParent(ChangeCommand command) {
    Parent parent = parentRepository.getById(command.parentId());

    parent.change(command);
    childService.replaceChildren(parent, command.children());

    applicationEventPublisher.publishEvent(ParentChangedEvent.from(parent));
}
```

이 구조의 장점은 명확합니다.

```text
DB rollback
↓
AFTER_COMMIT listener not invoked
↓
Kafka event not published
```

rollback된 변경이 Kafka로 나가는 위험을 줄일 수 있습니다.

또한 service layer에서 Kafka client detail을 분리할 수 있습니다.

도메인 변경과 이벤트 발행 trigger가 느슨하게 분리됩니다.

이 정도만 보면 꽤 괜찮은 구조처럼 보입니다.

하지만 운영에서 중요한 질문은 아직 남아 있습니다.

## 4. 남는 문제: commit 이후 실패

`AFTER_COMMIT`은 이름 그대로 commit 이후입니다.

그러면 이런 상황이 가능합니다.

```text
DB transaction commit success
↓
AFTER_COMMIT listener invoked
↓
Kafka publish failed
```

원인은 여러 가지일 수 있습니다.

```text
Kafka broker unavailable
producer timeout
serialization failure
message too large
process crash
deployment termination
network partition
```

이때 DB transaction은 이미 끝났습니다.

listener에서 예외가 발생해도 이미 commit된 DB 변경을 rollback할 수 없습니다.

즉 `@TransactionalEventListener(AFTER_COMMIT)`는 다음을 보장하지 않습니다.

```text
DB commit과 Kafka publish의 atomic commit
Kafka publish 실패 시 자동 재시도
commit된 변경에 대한 event delivery guarantee
process crash 이후 event 복구
```

이 지점이 중요합니다.

`@TransactionalEventListener`를 쓰면 transaction 문제를 해결한 것처럼 보이지만, 실제로는 실패 지점이 commit 이후로 이동한 것입니다.

물론 이것도 의미는 있습니다.

rollback event를 막는 것은 큰 개선입니다.

하지만 commit 이후 발행 실패를 놓치면 downstream consistency 문제는 여전히 남습니다.

## 5. Kafka transaction으로 해결할 수 있나

Spring Kafka는 Kafka transaction을 지원합니다.

또 JDBC transaction과 Kafka transaction을 동기화하는 구성도 가능합니다.

하지만 이것을 "DB와 Kafka의 완전한 distributed transaction"처럼 이해하면 위험합니다.

현실적으로 봐야 할 것은 commit ordering과 failure handling입니다.

```text
DB first
1. DB commit
2. Kafka commit

Kafka first
1. Kafka commit
2. DB commit
```

어느 쪽이든 중간 실패를 생각해야 합니다.

```text
DB commit success
Kafka commit failure

Kafka commit success
DB commit failure
```

Spring Kafka 문서에서도 transaction manager를 함께 쓸 수 있는 방법을 설명하지만, primary transaction commit 이후 동기화된 transaction commit이 실패하는 경우 애플리케이션이 보상 조치를 해야 한다는 점을 분리해서 봐야 합니다.

따라서 요구사항이 "event가 누락되면 안 된다"라면 Kafka transaction만으로 사고를 닫기보다는 Outbox를 먼저 검토하는 쪽이 낫습니다.

## 6. Outbox로 바꾸는 기준

Outbox의 핵심은 간단합니다.

DB 변경과 이벤트 저장을 같은 DB transaction 안에 넣습니다.

```text
@Transactional
↓
business table update
outbox table insert
↓
DB commit
```

그 다음 별도 publisher가 outbox row를 읽어 Kafka로 발행합니다.

```text
outbox table
↓
publisher job / relay
↓
Kafka
↓
mark published
```

이렇게 하면 commit된 변경에 대한 event intent가 DB에 남습니다.

Kafka publish가 실패해도 재시도할 수 있습니다.

```text
DB commit success
Kafka publish failed
↓
outbox row remains
↓
retry
```

Outbox를 검토해야 하는 신호는 명확합니다.

```text
Kafka event 누락이 downstream consistency 문제로 이어진다
manual replay가 자주 필요하다
AFTER_COMMIT listener 실패를 관측하거나 의심했다
배포/재시작 중 event loss를 허용할 수 없다
consumer가 eventual consistency를 전제로 동작한다
```

반대로 모든 이벤트에 Outbox가 필요한 것은 아닙니다.

예를 들어 analytics나 best-effort notification처럼 일부 누락을 허용할 수 있다면 `@TransactionalEventListener`와 retry/logging만으로 충분할 수 있습니다.

중요한 것은 이벤트의 성격을 분리하는 것입니다.

```text
state propagation event
→ Outbox 검토

best-effort side effect
→ TransactionalEventListener + retry/logging 가능
```

## 7. Outbox schema에서 봐야 할 것

Outbox는 개념은 단순하지만 운영 설계가 필요합니다.

최소한 다음 정보는 필요합니다.

```text
event_id
aggregate_type
aggregate_id
event_type
payload
status
created_at
published_at
retry_count
last_error
```

idempotency를 위해 event id는 안정적으로 만들어야 합니다.

consumer도 중복 수신을 전제로 설계해야 합니다.

Outbox publisher는 at-least-once에 가깝게 동작하는 경우가 많습니다.

```text
Kafka publish success
↓
mark published failed
↓
publisher retries
↓
duplicate event can be published
```

따라서 consumer는 다음 중 하나를 가져야 합니다.

```text
event_id 기반 deduplication
aggregate version check
idempotent upsert
last processed offset / event store
```

Outbox를 넣는 순간 문제가 사라지는 것이 아닙니다.

문제의 형태가 바뀝니다.

```text
event loss
→ retry / duplicate / ordering / cleanup 문제
```

그래도 state propagation event에서는 이 편이 낫습니다.

누락은 발견하기 어렵지만, 중복은 설계로 흡수하기 쉽기 때문입니다.

## 8. Claim-and-Check가 필요한 순간

Outbox는 delivery intent를 안전하게 남기는 패턴입니다.

하지만 payload size 문제를 직접 해결하지는 않습니다.

Kafka 이벤트 본문이 계속 커지는 구조라면 Outbox table에도 큰 payload가 쌓입니다.

```text
parent changed
↓
all child payload included
↓
outbox payload large
↓
Kafka message large
```

이 경우에는 Claim-and-Check를 같이 봐야 합니다.

Claim-and-Check는 Kafka에는 본문 전체가 아니라 claim만 싣는 방식입니다.

```json
{
  "eventId": "01J...",
  "aggregateId": "parent-123",
  "payloadRef": "s3://bucket/events/01J....json",
  "payloadHash": "sha256:...",
  "schemaVersion": 3
}
```

본문은 별도 저장소에 둡니다.

```text
DB transaction
↓
outbox row contains payload reference
↓
publisher sends claim event to Kafka
↓
consumer fetches payload by reference
```

이 방식은 Kafka message size 문제를 구조적으로 줄입니다.

하지만 비용도 있습니다.

```text
payload storage 운영
TTL / retention 정책
consumer fetch retry
payload checksum 검증
권한 / 암호화 / 개인정보 처리
payloadRef와 outbox row의 정합성
```

따라서 Claim-and-Check는 "멋진 아키텍처"라서 쓰는 것이 아니라, payload 크기가 event broker의 책임을 벗어날 때 쓰는 것입니다.

## 9. 선택지 비교

상황별 선택지를 이렇게 나눌 수 있습니다.

| 선택지 | 장점 | 한계 | 맞는 상황 |
|---|---|---|---|
| service 안에서 바로 Kafka publish | 단순함 | rollback event, dual-write 위험 | 중요도 낮은 side effect |
| `@TransactionalEventListener(AFTER_COMMIT)` | rollback event 방지 | commit 이후 publish 실패 복구 안 됨 | best-effort 이벤트 |
| Kafka transaction 동기화 | Kafka transaction boundary 활용 | commit ordering과 보상 처리 필요 | 제한된 범위의 transaction coordination |
| Outbox | event intent를 DB commit과 함께 저장 | publisher, 중복, cleanup 운영 필요 | state propagation event |
| Outbox + Claim-and-Check | delivery와 payload size를 분리 | 저장소/조회/TTL 비용 증가 | payload가 크고 누락도 허용 불가 |

이번 사례에서 제가 보는 기준은 이렇습니다.

```text
이벤트 누락을 허용할 수 있는가
→ 예: TransactionalEventListener로 충분할 수 있음
→ 아니오: Outbox 검토

payload가 Kafka 메시지로 계속 커지는가
→ 예: Claim-and-Check 검토
→ 아니오: Outbox payload로 시작 가능
```

## 10. 테스트에서 확인할 것

이 패턴은 단위 테스트만으로는 부족합니다.

특히 실패 지점을 강제로 만들어야 합니다.

확인해야 할 테스트는 다음과 같습니다.

```text
1. DB rollback 시 Kafka publish가 호출되지 않는가
2. DB commit 이후 listener가 호출되는가
3. AFTER_COMMIT listener 실패 시 DB 변경은 남는가
4. Kafka publish 실패 시 outbox row가 남는가
5. publisher retry 후 event가 발행되는가
6. publish 성공 후 mark published 실패 시 중복 발행을 consumer가 흡수하는가
7. 같은 aggregate의 event ordering이 깨지지 않는가
8. 큰 payload는 Kafka event가 아니라 payloadRef로 전달되는가
```

특히 3번은 꼭 봐야 합니다.

많은 코드가 listener에서 예외가 나면 전체 transaction이 rollback될 것처럼 착각합니다.

하지만 `AFTER_COMMIT`은 이미 commit 이후입니다.

이 차이를 테스트로 고정해두는 편이 좋습니다.

## 11. 체크리스트

DB 변경 이후 Kafka 이벤트를 발행한다면 다음 순서로 보겠습니다.

```text
1. 이벤트가 state propagation인지 best-effort side effect인지 구분했는가
2. rollback된 DB 변경에 대한 event 발행을 막았는가
3. commit 이후 Kafka publish 실패를 어떻게 복구할지 정했는가
4. AFTER_COMMIT listener 예외를 관측하고 alerting하는가
5. event_id와 aggregate_id가 있는가
6. consumer idempotency 전략이 있는가
7. Outbox publisher의 retry와 backoff가 있는가
8. publish 성공 / mark published 실패 중복 케이스를 처리하는가
9. payload 크기가 aggregate child count에 선형으로 증가하는가
10. Claim-and-Check 전환 기준을 정했는가
```

이 체크리스트의 핵심은 하나입니다.

트랜잭션 이후 이벤트 발행은 "언제 호출할 것인가"보다 "실패했을 때 무엇이 남는가"가 더 중요합니다.

## 12. 정리

`@TransactionalEventListener`는 좋은 도구입니다.

특히 `AFTER_COMMIT`은 rollback된 변경에 대한 이벤트 발행을 막는 데 유용합니다.

하지만 이것만으로 DB와 Kafka 사이의 dual-write 문제가 사라지지는 않습니다.

commit 이후 Kafka publish가 실패하면 DB에는 변경이 있고 Kafka에는 이벤트가 없는 상태가 가능합니다.

이벤트 누락이 downstream consistency 문제로 이어진다면 Outbox가 필요합니다.

payload가 계속 커져 Kafka 메시지 크기 문제까지 만든다면 Claim-and-Check까지 봐야 합니다.

이번 사례의 교훈은 간단합니다.

> `@TransactionalEventListener`는 이벤트 발행 시점을 transaction phase에 맞추는 도구이고, Outbox는 commit된 event intent를 복구 가능하게 남기는 도구다. 둘을 같은 문제의 해법처럼 섞어 보면 안 된다.

좋은 이벤트 발행 구조는 실패하지 않는 구조가 아닙니다.

실패했을 때 무엇을 재시도할 수 있는지 분명한 구조입니다.

## References

- [Spring Framework, Transaction-bound Events][spring-transaction-bound-events].
- [Spring Framework Javadoc, TransactionalEventListener][spring-transactional-event-listener].
- [Spring Kafka, Transactions][spring-kafka-transactions].
- [Microservices.io, Transactional Outbox][transactional-outbox].
- [Microservices.io, Transaction Log Tailing][transaction-log-tailing].
- [Mary Shaw, ICSE 2003][shaw-icse-2003].

[spring-transaction-bound-events]: https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html
[spring-transactional-event-listener]: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionalEventListener.html
[spring-kafka-transactions]: https://docs.spring.io/spring-kafka/reference/kafka/transactions.html
[transactional-outbox]: https://microservices.io/patterns/data/transactional-outbox.html
[transaction-log-tailing]: https://microservices.io/patterns/data/transaction-log-tailing.html
[shaw-icse-2003]: https://www.cs.cmu.edu/~Compose/shaw-icse03.pdf
