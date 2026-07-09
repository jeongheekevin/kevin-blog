---
title: "Kafka 메시지가 1MB를 넘을 때 LZ4 압축만으로 충분한가"
description: "Kafka RecordTooLargeException이 반복 발생했을 때 LZ4 압축과 max.request.size의 역할을 분리해 보고, 메시지 크기 테스트와 이벤트 모델 개선 필요성을 정리했습니다."
category: "Backend Engineering"
pubDate: "2026-03-20T00:00:00+09:00"
---

## TL;DR

Kafka producer에서 `RecordTooLargeException`이 반복 발생했습니다.

원인은 단순했습니다.

```text
Kafka producer default max.request.size
1MB

실제 메시지 크기
약 1.08MB ~ 2.63MB
평균 약 1.8MB
```

결과적으로 일부 대형 이벤트가 Kafka로 전송되지 않았습니다.

단기 조치는 두 가지였습니다.

```text
compression.type = lz4
max.request.size = 3MB
```

적용 후 반복되던 `RecordTooLargeException`은 0건으로 줄었습니다.

다만 두 설정의 역할은 같지 않습니다.

`compression.type=lz4`는 broker로 전달되는 record batch 크기를 실제로 줄입니다.

반면 `max.request.size`는 메시지를 작게 만들지 않습니다. producer가 압축 전 record batch를 어디까지 만들고 보낼 수 있는지를 제한하는 client-side guardrail에 가깝습니다.

하지만 이 글의 결론은 "Kafka 설정을 올리면 된다"가 아닙니다.

핵심은 따로 있습니다.

> 메시지 크기가 비즈니스 객체의 child count에 선형으로 증가한다면, LZ4 압축은 좋은 단기 완화책이다. `max.request.size`는 그 과정에서 producer의 압축 전 batch 상한을 맞추는 설정이지, 메시지 모델을 해결하는 설정은 아니다. 장기적으로는 delta event, event 분리, 또는 Claim-and-Check 패턴으로 메시지 모델을 바꿔야 한다.

## 1. 문제 상황

어떤 변경 이벤트를 Kafka로 발행하고 있었습니다.

단순화하면 다음과 같은 이벤트입니다.

```text
parent entity changed
↓
include all child entities in event payload
↓
publish to Kafka
↓
downstream consumers refresh state
```

평소에는 문제가 없었습니다.

하지만 child entity가 매우 많은 parent entity에서 변경이 발생하면 메시지 크기가 커졌습니다.

구조상 당연한 결과였습니다.

```text
payload size
≈
parent metadata
+
child entity payload * child count
```

child count가 늘수록 메시지 크기가 선형으로 증가했습니다.

결국 Kafka producer 기본 한도인 1MB를 넘는 메시지가 발생했고, 전송이 실패했습니다.

```text
RecordTooLargeException
```

운영 영향은 단순히 "로그에 에러가 난다"가 아니었습니다.

변경 이벤트가 Kafka로 나가지 않으면 downstream consumer는 변경을 모릅니다.

```text
source state changed
↓
Kafka publish failed
↓
downstream state stale
↓
serving / cache / search index mismatch
```

이런 문제는 빠르게 막아야 했습니다.

## 2. 먼저 확인한 것: 실제 메시지 크기

설정을 바꾸기 전에 먼저 메시지 크기를 봤습니다.

에러 로그에서 실패한 메시지 크기를 샘플링했습니다.

결과는 다음과 같았습니다.

```text
sampled message size
1.08MB ~ 2.63MB

average
약 1.8MB
```

이 수치가 중요합니다.

왜냐하면 Kafka size 문제는 무턱대고 한도를 크게 올리면 안 되기 때문입니다.

확인해야 할 것이 많습니다.

```text
producer max.request.size
broker max.message.bytes
topic별 broker 설정
consumer fetch.max.bytes
replica fetch 설정
network overhead
compression 후 크기
```

여기서 중요한 점은 각 설정이 보는 크기가 다르다는 것입니다.

`max.request.size`는 producer request 크기 제한이며, 사실상 압축 전 record batch 크기의 상한으로 동작합니다.

반면 broker의 `message.max.bytes`나 topic의 `max.message.bytes`는 압축이 켜져 있다면 압축 후 record batch 크기를 기준으로 봅니다.

따라서 같은 `RecordTooLargeException`이라도 실패 지점은 나뉩니다.

```text
producer-side failure
serialized record batch > max.request.size

broker-side failure
compressed record batch > message.max.bytes / max.message.bytes
```

이번 조치의 효과를 해석할 때도 이 구분이 필요합니다.

LZ4는 broker가 받는 batch 크기와 네트워크 전송량을 줄였습니다.

`max.request.size=3MB`는 압축 전 record batch가 producer의 기본 1MB guardrail에 걸리지 않도록 만든 설정입니다.

## 3. 선택지 비교

가능한 선택지는 여러 가지였습니다.

| 선택지 | 장점 | 한계 | 판단 |
|---|---|---|---|
| LZ4 압축 | broker로 가는 record batch 크기를 실제로 줄임 | 압축 후에도 원본 모델은 커짐 | 단기 핵심 |
| `max.request.size` 증가 | producer의 압축 전 batch 상한을 통과시킴 | 메시지를 작게 만들지는 않음 | 보조 guardrail |
| broker/topic 한도 증가 | 더 큰 메시지 허용 가능 | cluster 전체 영향 검토 필요 | 이번에는 불필요 |
| delta event | payload 자체를 줄임 | consumer 모델 변경 필요 | 장기 개선 |
| Claim-and-Check | Kafka에는 claim만 싣고 본문은 별도 저장 | 저장소/조회 경로 필요 | 구조 개선 |

단기적으로는 압축을 핵심 완화책으로 두고, producer 상한을 관측된 원본 batch 크기에 맞췄습니다.

하지만 장기적으로는 메시지 모델을 바꿔야 한다고 봤습니다.

## 4. 왜 LZ4였나

이번 payload는 반복 구조를 가진 JSON이었습니다.

이런 payload는 압축이 잘 먹힙니다.

```text
same field names
similar nested structure
repeated child entity shape
```

압축 알고리즘은 여러 가지가 있습니다.

```text
gzip
snappy
lz4
zstd
```

실시간 이벤트 전송에서는 압축률만 보면 안 됩니다.

CPU 비용과 latency도 봐야 합니다.

LZ4는 압축률과 속도 사이에서 균형이 좋습니다.

```text
compression.type = lz4
```

이번 payload처럼 반복 JSON 구조가 많으면 LZ4 압축 효과를 기대할 수 있었습니다.

다만 "압축이 잘 됐다"는 말은 압축률로 확인해야 합니다.

운영에서 남겨야 하는 수치는 최소한 다음 세 가지입니다.

```text
serialized payload size before compression
compressed record batch size after compression
producer/broker에서 실제로 실패한 limit
```

이 글에 남긴 수치는 실패한 메시지의 원본 크기 분포와 에러 발생 건수입니다.

```text
원본 메시지 크기: 약 1.08MB ~ 2.63MB
RecordTooLargeException: 일 평균 약 5건 → 0건
```

반면 압축 전후 byte 차이는 별도로 남기지 못했습니다.

따라서 이 사례에서 정확한 표현은 "LZ4가 큰 폭으로 줄였다"가 아니라, "반복 JSON 구조에서는 LZ4가 유효한 완화책이 될 수 있고, 실제 기여도는 압축 전후 byte를 함께 봐야 한다"에 가깝습니다.

다만 여기서 중요한 점이 있습니다.

압축은 payload를 줄여주지만, payload 모델 자체를 바꾸지는 않습니다.

```text
bad model compressed
 still bad model
```

압축은 좋은 완화책이지만, 설계 개선을 대체하지는 않습니다.

## 5. max.request.size가 실제로 한 일

샘플링한 실패 메시지 최대 크기는 약 2.63MB였습니다.

그래서 producer의 압축 전 batch 상한은 3MB로 잡았습니다.

```text
max.request.size = 3MB
```

이 값은 무작정 크게 잡은 값이 아닙니다.

```text
observed max: 약 2.63MB
configured limit: 3MB
```

현재 관측된 원본 record batch를 producer가 만들 수 있도록 둔 최소한의 여유입니다.

하지만 이 값은 broker 부담을 줄이지 않습니다.

오히려 producer가 더 큰 압축 전 batch를 만들 수 있게 허용합니다.

broker가 실제로 받는 크기를 줄인 것은 LZ4 압축입니다.

따라서 결과를 이렇게 해석하는 편이 더 정확합니다.

```text
max.request.size
producer-side uncompressed batch guardrail 통과

compression.type=lz4
broker-side compressed batch size 감소
```

물론 3MB도 영구적인 답은 아닙니다.

child count가 더 늘어나면 다시 넘을 수 있습니다.

그래서 이 값은 "구조 개선 전까지 운영 장애를 막는 guardrail"에 가깝습니다.

이 guardrail이 유지 가능한지는 child count 분포로 판단해야 합니다.

```text
현재 p95 / p99 child count
최대 child count 증가 속도
child 1개당 평균 payload 증가량
3MB에 도달하는 예상 child count
```

예를 들어 child count가 1,000 이상으로 늘어나는 비즈니스 요구가 이미 보인다면, `max.request.size=3MB`는 오래 버틸 설정이 아닙니다.

그 경우에는 설정 상향보다 event model 변경 계획을 먼저 잡아야 합니다.

## 6. 테스트는 메시지 크기를 직접 밟아야 한다

이런 문제는 단위 테스트로 DTO 생성만 확인하면 부족합니다.

실제로 Kafka producer가 해당 크기의 메시지를 보낼 수 있는지 확인해야 합니다.

그래서 메시지 크기 구간을 나눠 테스트했습니다.

```text
1.3MB message
2.6MB message
3MB+ message
```

테스트 목적은 세 가지였습니다.

```text
1. 기존 1MB 제한을 넘는 메시지가 재현되는가
2. 설정 변경 후 관측된 최대 크기 수준의 메시지가 전송되는가
3. 한도 초과 케이스가 의도대로 실패하는가
```

여기서 중요한 것은 실패 지점을 분리하는 것입니다.

같은 `RecordTooLargeException`이어도 producer가 압축 전 batch를 만들다가 실패한 것인지, broker가 압축 후 batch를 거절한 것인지에 따라 처방이 달라집니다.

그래서 테스트에서는 성공 여부만 볼 것이 아니라, 실패한 설정값과 에러 메시지도 함께 확인해야 합니다.

## 7. 적용 결과

적용 후 반복되던 전송 실패는 사라졌습니다.

```text
Before
RecordTooLargeException 일 평균 약 5건

After
RecordTooLargeException 0건
```

이 수치만으로는 전체 이벤트 품질을 모두 설명하지 못합니다.

일 평균 5건이 큰 문제인지 판단하려면 분모가 필요합니다.

```text
전체 발행 이벤트 수
대형 이벤트 비율
실패한 이벤트가 downstream state에 미친 영향
재시도 또는 보정 경로 존재 여부
```

예를 들어 전체 이벤트가 하루 수백만 건이라면 비율로는 작아 보일 수 있습니다.

하지만 실패한 이벤트가 대형 parent entity에 집중되어 downstream state를 오래 stale하게 만든다면 운영 영향은 작지 않습니다.

그래서 이 개선은 "전체 Kafka traffic을 크게 바꾼 개선"이라기보다, "특정 대형 payload에서 발생하던 publish failure를 제거한 안정화"로 해석하는 편이 맞습니다.

운영 관점에서 이 결과는 중요합니다.

```text
large entity change
↓
Kafka publish succeeds
↓
downstream consumer receives event
↓
state refresh path restored
```

하지만 다시 말하지만, 이것은 단기 안정화입니다.

이번 조치로 장애는 막았지만, 메시지 모델의 근본 위험은 남아 있습니다.

## 8. 진짜 문제: payload가 child count에 비례한다

이 사례의 본질은 Kafka 설정이 아닙니다.

이벤트 payload가 child entity 수에 비례해 커지는 모델입니다.

```text
child count 100
→ medium payload

child count 500
→ large payload

child count 1000
→ too large payload
```

이런 구조에서는 언젠가 다시 한도를 만납니다.

한도를 3MB로 올리면 3MB까지는 버팁니다.

그 다음에는 5MB, 10MB 이야기가 나올 수 있습니다.

하지만 그 방향은 위험합니다.

Kafka는 큰 파일을 옮기는 시스템이 아닙니다.

Kafka는 event log입니다.

따라서 이벤트에는 필요한 변화 신호를 담고, 큰 상태 본문은 다른 방식으로 다루는 편이 낫습니다.

## 9. 장기 개선 방향

장기적으로는 세 가지 방향을 검토할 수 있습니다.

### 9.1 Delta event

전체 child list를 보내지 않고 변경된 child만 보냅니다.

```text
Before
parent changed + all children

After
parent changed + changed children only
```

장점은 payload가 작아진다는 점입니다.

단점은 consumer가 이전 상태와 delta를 합치는 책임을 가져야 한다는 점입니다.

운영 비용은 consumer state 관리입니다.

```text
event ordering
idempotency
deduplication
replay 시 최종 상태 재구성
consumer별 state version 관리
```

### 9.2 Event split

parent change event와 child change event를 분리합니다.

```text
parent.changed
child.changed
```

이렇게 하면 event boundary가 더 명확해집니다.

하지만 consumer가 여러 event stream을 조합해야 할 수 있습니다.

운영 비용은 event choreography입니다.

```text
parent/child event 순서
consumer join logic
partial update 허용 범위
event schema versioning
```

### 9.3 Claim-and-Check

Kafka에는 큰 payload를 직접 넣지 않고, payload를 조회할 수 있는 claim만 넣습니다.

```text
large payload
→ object storage / database

Kafka event
→ payload id / version / checksum
```

consumer는 Kafka event를 받은 뒤 필요한 시점에 본문을 조회합니다.

이 방식은 메시지 크기 문제를 구조적으로 줄입니다.

대신 저장소, TTL, idempotency, read consistency를 함께 설계해야 합니다.

운영 비용은 외부 저장소와 조회 경로입니다.

```text
payload storage TTL
consumer fetch retry
object versioning
checksum 검증
Kafka event와 payload 저장의 원자성
```

## 10. 체크리스트

Kafka에서 `RecordTooLargeException`을 만나면 다음 순서로 보겠습니다.

```text
1. 실패 메시지 크기 분포를 샘플링했는가
2. 평균과 최대값을 분리해서 봤는가
3. producer max.request.size와 broker max.message.bytes를 모두 확인했는가
4. consumer fetch 설정도 영향 범위에 포함했는가
5. payload가 특정 entity count에 선형으로 증가하는가
6. 반복 JSON 구조라면 compression 효과를 측정했는가
7. 메시지 크기별 integration test를 작성했는가
8. 실패 지점이 producer-side인지 broker-side인지 구분했는가
9. 한도 증가는 임시 대응이라고 명시했는가
10. delta event 또는 Claim-and-Check 전환 기준을 정했는가
```

Kafka size 문제는 설정값 하나만 보고 끝내면 안 됩니다.

producer, broker, consumer, event model을 같이 봐야 합니다.

## 11. 정리

이번 문제는 `RecordTooLargeException`으로 시작했습니다.

하지만 본질은 producer 설정이 아니라 event payload 설계였습니다.

단기적으로는 LZ4 압축으로 broker가 받는 batch 크기를 줄이고, `max.request.size=3MB`로 producer의 압축 전 batch 상한을 맞춰 전송 실패를 제거했습니다.

```text
RecordTooLargeException
일 평균 약 5건 → 0건
```

하지만 장기적으로는 다음 질문을 남겨야 합니다.

```text
이 이벤트가 정말 전체 child payload를 담아야 하는가
변경분만 보내면 안 되는가
Kafka에는 claim만 싣고 본문은 별도 저장하면 안 되는가
```

이번 사례의 교훈은 간단합니다.

> Kafka 메시지 크기 문제는 압축 전/후 크기와 실패 지점을 분리해서 보고, 마지막에는 event model 검토로 끝나야 한다.

압축은 운영을 살리는 좋은 응급처치입니다.

`max.request.size` 증가는 메시지를 줄이는 처방이 아니라 producer 쪽 guardrail 조정입니다.

하지만 payload가 계속 커지는 구조라면, 언젠가는 메시지 모델을 바꿔야 합니다.

## References

- [Apache Kafka Producer Configs][kafka-producer-configs].
- [Apache Kafka Broker Configs][kafka-broker-configs].
- [Apache Kafka Topic Configs][kafka-topic-configs].
- [Mary Shaw, ICSE 2003][shaw-icse-2003].

[kafka-producer-configs]: https://kafka.apache.org/41/configuration/producer-configs/
[kafka-broker-configs]: https://kafka.apache.org/41/configuration/broker-configs/
[kafka-topic-configs]: https://kafka.apache.org/41/configuration/topic-configs/
[shaw-icse-2003]: https://www.cs.cmu.edu/~Compose/shaw-icse03.pdf
