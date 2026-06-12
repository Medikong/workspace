---
id: TROUBLE-007
title: "Kafka producer trace 계측이 호출부에 흩어지는 문제"
status: triaged
priority: p2
severity: medium
area: observability
repos:
  - workspace
  - service
owner: unassigned
created: 2026-06-12
updated: 2026-06-12
resolved: null
tags:
  - kafka
  - tracing
  - producer
  - outbox
  - api-design
related:
  - TROUBLE-005
  - TROUBLE-006
  - docs/architecture/observability/tracing/README.md
links:
  - https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/aiokafka/aiokafka.html
  - https://aiokafka.readthedocs.io/en/stable/producer.html
---

# Kafka producer trace 계측이 호출부에 흩어지는 문제

## Context

payment-service의 outbox dispatcher는 `/payments` 요청 시점의 trace context를 `payment_events.trace_context`에 저장하고, dispatcher가 outbox row를 Kafka로 발행할 때 header로 복원한다. 이 덕분에 ticket-service와 notification-service의 consumer span은 같은 trace id에 붙을 수 있다.

다만 producer 발행 자체는 별도 span으로 자동 수집되지 않는다. 현재 repo는 `aiokafka` producer 자동 계측을 기본으로 사용하지 않고, `kafka_utils.build_producer_headers()`는 header 전파만 담당한다. 그래서 호출부가 `start_producer_span`, header 주입, `send_and_wait()` 호출 순서를 직접 맞춰야 한다.

이번 기록은 단일 bug라기보다 유지보수 리스크다. 지금처럼 service별 호출부가 raw `send_and_wait()`를 직접 부르면, 한 곳만 빠져도 Kafka publish span이 누락되고 trace 연결 품질이 서비스마다 달라질 수 있다.

## Symptoms

- 관찰된 현상:
  - `create_payment`에서 outbox row가 저장되는 구간과 Kafka consumer 구간은 trace로 확인된다.
  - outbox row를 기반으로 Kafka에 producer publish하는 구간은 Tempo에서 별도 producer span으로 보이지 않는다.
  - producer span을 보이게 하려면 호출부마다 `start_producer_span`과 header 주입을 직접 추가해야 한다.
- 재현 조건:
  - background outbox dispatcher처럼 원 HTTP request context 밖에서 `AIOKafkaProducer.send_and_wait()`를 호출한다.
  - 저장된 trace carrier를 현재 OpenTelemetry context로 복원하지 않는다.
  - raw producer API를 직접 호출하거나 header 생성만 수행한다.
- 기대 동작:
  - Kafka publish는 기본적으로 `messaging.system=kafka`, `messaging.operation=publish` 속성을 가진 producer span을 남긴다.
  - outbox dispatcher는 저장된 trace context를 parent로 복원한 뒤 producer span을 만들고, 그 producer span context를 Kafka header로 전달한다.
  - 호출부는 raw producer와 거의 같은 `send_and_wait()` API를 쓰면서도 trace 계측 누락을 만들기 어렵다.
- 실제 동작:
  - raw `send_and_wait()`는 trace span을 만들지 않는다.
  - `build_producer_headers()`만으로는 consumer 연결은 가능하지만 producer 발행 단계가 trace 상세 화면에서 비어 보일 수 있다.
  - 계측 규칙이 호출부 관례에 남아 있어 새 producer 호출이 생길 때마다 반복 구현이 필요하다.

## Impact

- 영향 범위:
  - payment-service outbox dispatcher의 Kafka publish 관측.
  - ticket-service, reservation-service처럼 Kafka producer를 직접 쓰는 서비스.
  - 신규 이벤트 발행 코드의 trace 품질과 운영 조사 경험.
- 우선 처리 이유:
  - Kafka publish는 DB commit과 consumer 처리 사이의 중요한 경계다.
  - producer span이 없으면 outbox 지연, broker 전송 지연, consumer 지연을 trace 화면에서 분리해 보기 어렵다.
  - 호출부마다 수동 계측하면 한 번의 누락이 조용히 운영 관측 품질 차이로 이어진다.
- 우회 방법:
  - 단기적으로는 각 producer 호출부를 찾아 `start_producer_span`과 header 주입을 직접 추가한다.
  - 이 방식은 작동하지만 반복이 많고, raw `send_and_wait()` 사용을 막지 못한다.

## Current Code Shape

현재는 Kafka에 메시지를 보낼 때마다 호출부가 producer span과 header 주입을 함께 조립해야 추적이 이어진다. 아래처럼 `start_producer_span()`을 열고, 그 안에서 `send_and_wait()`를 호출하며, header도 별도 helper로 만들어야 한다.

```python
event_type = PaymentEventType(event.event_type)
topic = _payment_event_topic(event_type)
trace_carrier = _trace_carrier(event.trace_context)
try:
    with start_producer_span(
        topic,
        carrier=trace_carrier,
        attributes={"payment.event_type": event.event_type},
    ):
        await kafka_producer.send_and_wait(
            topic,
            event.payload,
            headers=_producer_headers(
                correlation_id=event.payload.get("correlationId"),
                fallback_carrier=trace_carrier,
            ),
        )
except (KafkaError, RuntimeError) as exc:
    self._mark_failed(event, exc)
    self._telemetry.record(
        PaymentEventPublishRecorded(event_type=event_type, result=MetricResult.FAILURE)
    )
    set_current_span_attributes({"payment.outbox.publish.result": "failure"})
    raise
```

이 코드는 동작하지만 계측 규칙이 호출부에 드러나 있다. 새 Kafka producer 호출을 추가할 때 이 span wrapper나 header 복원 중 하나라도 빠지면, 메시지는 발행되더라도 Tempo에서는 producer 발행 구간이 보이지 않거나 consumer span이 기대한 parent 아래에 붙지 않을 수 있다. 따라서 같은 규칙을 매번 손으로 붙이는 대신, producer wrapper의 `send_and_wait()` 안으로 넣어 누락 가능성을 줄이는 방향이 필요하다.

## Investigation

| 시간 | 확인 내용 | 결과 |
| --- | --- | --- |
| 2026-06-12 | `kafka-utils` 확인 | `build_producer_headers()`는 `traceparent`, `tracestate`, `correlation_id` header 생성만 담당함 |
| 2026-06-12 | payment outbox dispatcher 확인 | 저장된 `trace_context`를 Kafka header로 복원하지만 producer publish span은 별도 보장되지 않음 |
| 2026-06-12 | consumer 계측 확인 | `start_consumer_span(message)`는 Kafka header에서 parent context를 추출해 consumer span을 시작함 |
| 2026-06-12 | OpenTelemetry aiokafka 계측 확인 | `opentelemetry-instrumentation-aiokafka`가 존재하지만 outbox DB에 저장된 trace context를 자동으로 알 수는 없음 |
| 2026-06-12 | aiokafka producer API 확인 | `send()`와 `send_and_wait()`에 header를 넘기는 구조이며, 앱 레벨 interceptor/hook을 표준 확장점처럼 쓰는 형태는 확인하지 못함 |
| 2026-06-12 | 설계 방향 논의 | raw producer를 감싸되 `send_and_wait()` 이름과 기존 인자를 유지하고, 추가 trace 값은 함수형 옵션 패턴으로 받는 방향으로 정리 |

## Decision

Kafka producer 계측은 호출부마다 직접 조립하지 않는다. `packages/kafka-utils`가 trace-aware producer wrapper를 제공하고, 서비스 composition root가 raw `AIOKafkaProducer` 대신 wrapper를 주입하는 방향으로 정리한다.

- wrapper는 새 이름의 publish API를 강요하지 않는다.
- 기본 메서드는 기존 `AIOKafkaProducer.send_and_wait(topic, value, *, key=None, partition=None, timestamp_ms=None, headers=None)`와 같은 이름과 인자 모양을 유지한다.
- 추가로 필요한 값은 Go의 함수형 옵션 패턴처럼 선택 옵션으로 받는다.
- 옵션 예시는 다음과 같다.
  - `with_trace_context(trace_context)` 또는 `with_trace_carrier(carrier)`
  - `with_correlation_id(correlation_id)`
  - `with_span_attributes({...})`
  - `with_span_name("kafka.produce payment-approved")`
- wrapper 내부 책임은 다음과 같다.
  - 저장된 carrier가 있으면 parent context로 추출한다.
  - producer span을 `SpanKind.PRODUCER`로 시작한다.
  - producer span context를 Kafka header에 주입한다.
  - caller가 넘긴 header와 trace/correlation header를 충돌 없이 병합한다.
  - raw producer의 `send_and_wait()`를 호출한다.
  - 실패 시 span status와 exception 기록 방식을 일관되게 적용한다.
- outbox dispatcher는 저장된 trace context를 wrapper 옵션으로 넘긴다.
- 일반 HTTP 요청 내부 producer는 옵션 없이 호출해도 현재 OpenTelemetry context에서 header가 주입되게 한다.
- 공식 `opentelemetry-instrumentation-aiokafka`는 보조 안전망으로 검토한다.
  - raw producer 직접 사용을 잡는 데는 도움이 될 수 있다.
  - 그러나 outbox에 저장된 trace context 복원은 애플리케이션 도메인 규칙이므로 wrapper가 소유해야 한다.
  - wrapper와 공식 instrumentation을 동시에 켤 때 중복 producer span이 생기는지 별도 검증이 필요하다.

## Actions

| 상태 | 작업 | 담당 | 링크 |
| --- | --- | --- | --- |
| done | 현재 문제와 설계 결론을 트러블 문서로 기록 | workspace | 이 문서 |
| todo | `kafka-utils`에 trace-aware producer wrapper API 초안 작성 | service |  |
| todo | `send_and_wait()` 호환 인자와 함수형 옵션 타입 설계 | service |  |
| todo | payment-service outbox dispatcher를 wrapper 옵션 방식으로 변경 | service | `services/payment-service/app/services/payment_events.py` |
| todo | reservation-service와 ticket-service producer 호출부를 raw producer 직접 호출 없이 정리 | service |  |
| todo | `opentelemetry-instrumentation-aiokafka` 적용 가능성과 중복 span 여부 검증 | service |  |
| todo | wrapper가 누락되지 않도록 raw `AIOKafkaProducer.send_and_wait()` 직접 호출 검색 또는 테스트 규칙 추가 | service |  |

## Resolution

미해결. 현재 결론은 raw Kafka producer 호출부마다 trace span과 header 주입을 직접 붙이는 방식은 장기 유지보수에 맞지 않는다는 것이다.

닫기 위한 조건은 다음과 같다.

- 서비스 코드가 raw `AIOKafkaProducer.send_and_wait()`를 직접 호출하지 않는다.
- wrapper의 `send_and_wait()`만 사용해도 producer span과 trace header가 기본으로 남는다.
- outbox dispatcher는 DB에 저장된 trace context를 wrapper 옵션으로 넘기고, producer span이 원 요청 trace id 아래에 붙는다.
- 공식 aiokafka instrumentation을 켤지 말지 결정하고, 켠다면 중복 span이 생기지 않음을 테스트 또는 운영 trace로 확인한다.
