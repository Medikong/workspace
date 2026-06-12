# Kafka producer trace wrapper 설계

## 배경

payment-service는 outbox 패턴을 쓰기 때문에 HTTP 요청 안에서 바로 Kafka에 메시지를 보내지 않는다. `/payments` 요청 처리 중에는 현재 OpenTelemetry context가 살아 있지만, 실제 Kafka 발행은 나중에 background dispatcher가 `payment_events` row를 읽어 수행한다.

그래서 `payment_events.trace_context` 컬럼을 추가해 요청 시점의 `traceparent`, `tracestate`, `trace_id`, `span_id`를 outbox metadata로 보관했다. 이 방향은 맞다. 다만 저장된 trace context를 producer 발행 시점에 항상 복원하고, producer span과 Kafka header까지 일관되게 만들려면 현재 호출부 방식은 부족하다.

현재는 Kafka에 메시지를 보낼 때마다 아래처럼 producer span과 header 복원을 직접 조립해야 한다.

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

이 코드는 동작하지만 계측 규칙이 호출부에 흩어진다. 새 producer 호출을 추가할 때 `start_producer_span()`, 저장된 carrier 복원, header 주입 중 하나라도 빠지면 메시지는 발행되지만 Tempo에서 producer 발행 구간이 보이지 않거나 consumer span이 기대한 parent 아래에 붙지 않을 수 있다.

## 목표

- 서비스 코드가 raw `AIOKafkaProducer.send_and_wait()`를 직접 조립하지 않게 한다.
- 기존 `send_and_wait()` 이름과 기본 인자 모양은 유지한다.
- outbox처럼 저장된 trace context가 있는 경우 producer wrapper가 parent context를 복원한다.
- 일반 HTTP 요청 내부 producer 호출은 별도 옵션 없이 현재 OpenTelemetry context를 사용한다.
- trace header, correlation header, producer span 생성 규칙을 `packages/kafka-utils` 한 곳에 모은다.

## 제안 1. `send_and_wait()` 호환 producer wrapper

`packages/kafka-utils`에 raw `AIOKafkaProducer`를 감싸는 wrapper를 둔다. 이름은 새 도메인 API를 만들기보다 producer 역할을 그대로 드러내는 쪽이 좋다. 중요한 점은 호출부가 `publish_event()` 같은 새 함수를 외우지 않아도 된다는 것이다.

예상 API:

```python
producer = create_kafka_producer(settings.kafka_bootstrap_servers, client_id=settings.service_name)

await producer.send_and_wait(
    topic,
    payload,
    headers=headers,
)
```

trace 옵션이 필요한 경우:

```python
await producer.send_and_wait(
    topic,
    payload,
    with_trace_context(event.trace_context),
    with_correlation_id(event.payload.get("correlationId")),
    with_span_attributes({"payment.event_type": event.event_type}),
)
```

wrapper 내부 책임:

- raw producer의 `start()`, `stop()`, `send()`, `send_and_wait()`를 위임한다.
- `send_and_wait()` 안에서 producer span을 `SpanKind.PRODUCER`로 시작한다.
- span attribute에는 낮은 cardinality 값만 붙인다.
  - `messaging.system = kafka`
  - `messaging.destination.name = <topic>`
  - `messaging.operation = publish`
  - `event.type` 또는 `payment.event_type`
  - `correlation_id`
- producer span context를 Kafka header에 주입한다.
- caller가 넘긴 header와 wrapper가 만든 header를 병합한다.
- trace 관련 header는 wrapper가 만든 값을 우선한다.
- raw producer 호출 실패는 span status와 exception으로 기록한 뒤 그대로 전파한다.

이 제안의 효과:

- 새 Kafka 발행 코드에서 span 생성 누락 가능성이 줄어든다.
- 서비스별 producer 호출 모양이 같아진다.
- 호출부는 raw producer를 쓰는 것처럼 읽히지만 trace 전파 규칙은 공통 패키지가 소유한다.

## 제안 2. 함수형 옵션으로 trace/correlation 값을 주입

Python에서도 Go의 함수형 옵션 패턴과 비슷하게 추가 계측 값을 옵션 객체 또는 callable로 받을 수 있다. 기존 `send_and_wait(topic, value, *, headers=...)` 인자를 깨지 않고, 마지막 positional option이나 keyword option으로 확장한다.

옵션 후보:

```python
with_trace_context(trace_context)
with_trace_carrier(carrier)
with_correlation_id(correlation_id)
with_span_attributes(attributes)
with_span_name(name)
```

옵션 처리 기준:

- `with_trace_context(trace_context)`는 outbox row에 저장된 `{"carrier": {...}, "trace_id": "...", "span_id": "..."}` 형태를 받는다.
- `with_trace_carrier(carrier)`는 이미 정리된 W3C propagation carrier를 받는다.
- 둘 다 없으면 현재 OpenTelemetry context를 사용한다.
- `with_correlation_id()`는 Kafka header와 span attribute에 같이 반영한다.
- `with_span_attributes()`는 topic, messaging 기본 attribute 뒤에 병합한다.
- 고카디널리티 ID는 기본 span attribute로 넣지 않는다.
  - `payment_id`, `reservation_id`, `user_id`, `event_id`는 payload, audit log, structured log에서 찾는다.
  - trace에는 `event_type`, `topic`, `result`처럼 aggregation 가능한 값만 우선한다.

예상 타입 모양:

```python
KafkaProducerOption = Callable[[KafkaProducerOptions], None]

@dataclass
class KafkaProducerOptions:
    trace_carrier: Mapping[str, str] | None = None
    correlation_id: str | None = None
    span_name: str | None = None
    span_attributes: dict[str, str | int | float | bool] = field(default_factory=dict)
```

이 제안의 효과:

- outbox 호출부는 저장된 trace context만 넘기면 된다.
- 일반 producer 호출부는 옵션 없이도 현재 request span 아래에 producer span이 붙는다.
- 앞으로 retry, partition key, custom span name 같은 확장이 생겨도 wrapper signature를 계속 크게 바꾸지 않아도 된다.

## 제안 3. raw producer 직접 사용을 줄이고 검증 규칙을 추가

wrapper를 만들어도 서비스 코드가 raw `AIOKafkaProducer`를 직접 주입받으면 다시 누락이 생길 수 있다. 따라서 composition root와 테스트에서 raw producer 사용을 줄이는 규칙을 둔다.

적용 대상:

- `payment-service`
  - outbox dispatcher가 wrapper `send_and_wait()`를 사용한다.
  - `event.trace_context`는 `with_trace_context()`로 넘긴다.
  - retry 상태 갱신과 publish metric은 payment outbox 책임으로 유지한다.
- `reservation-service`
  - HTTP 요청 안에서 Kafka 이벤트를 발행하므로 별도 trace option 없이 current context 기반 producer span을 기대한다.
  - correlation id는 `with_correlation_id()`로 명시한다.
- `ticket-service`
  - `payment-approved` consumer span 안에서 `ticket-issued`를 발행한다.
  - current consumer span 아래에 producer span이 붙어야 한다.
  - correlation id는 수신 payload에서 이어받는다.

검증 규칙:

- 단위 테스트:
  - wrapper가 저장된 trace carrier를 parent context로 추출한다.
  - wrapper가 producer span 안에서 Kafka header를 주입한다.
  - caller header와 wrapper header 병합 우선순위가 고정된다.
  - `send_and_wait()` 실패 시 exception이 삼켜지지 않는다.
- 서비스 테스트:
  - payment outbox dispatcher는 raw `start_producer_span()`을 직접 호출하지 않는다.
  - payment outbox dispatcher는 wrapper option으로 `trace_context`를 넘긴다.
  - Kafka header에는 producer span context의 `traceparent`가 들어간다.
- 정적 검색 또는 lint성 테스트:
  - 서비스 코드에서 raw `AIOKafkaProducer.send_and_wait()` 직접 호출을 찾는다.
  - 허용 예외는 `packages/kafka-utils` wrapper 구현과 테스트 fake 정도로 제한한다.
- 운영 검증:
  - Tempo에서 `POST /payments` 아래에 `kafka.produce payment-approved`가 보인다.
  - 그 뒤에 `ticket-service kafka.consume payment-approved`가 같은 trace id로 이어진다.
  - `ticket-issued` 발행도 `kafka.produce ticket-issued`로 확인된다.

이 제안의 효과:

- trace 품질이 호출자 습관이 아니라 API 사용 방식으로 보장된다.
- outbox, HTTP request, consumer handler 안 producer가 같은 규칙을 공유한다.
- 나중에 `opentelemetry-instrumentation-aiokafka`를 켤 때도 중복 span 여부를 wrapper 중심으로 검증할 수 있다.

## 공식 aiokafka instrumentation 위치

`opentelemetry-instrumentation-aiokafka`는 보조 안전망으로 검토한다. 이 계측은 raw `AIOKafkaProducer` 호출을 자동으로 감싸는 데 도움이 될 수 있다. 다만 outbox row에 저장된 `trace_context`를 언제 parent context로 복원해야 하는지는 애플리케이션 규칙이다. 공식 instrumentation만으로는 DB에 저장된 outbox metadata의 의미를 알 수 없다.

따라서 기본 방향은 다음과 같다.

```text
필수: kafka-utils producer wrapper
필수: outbox trace_context option 복원
선택: opentelemetry-instrumentation-aiokafka 보조 안전망
```

공식 instrumentation을 켜는 경우에는 wrapper span과 자동 instrumentation span이 중복되지 않는지 먼저 확인한다.

## 구현 순서

1. `packages/kafka-utils`에 wrapper와 option 타입을 추가한다.
2. wrapper 테스트로 span 시작, header 주입, trace carrier 복원, 실패 전파를 고정한다.
3. payment-service outbox dispatcher를 파일럿 대상으로 wrapper option 방식으로 바꾼다.
4. payment-service 단위 테스트와 synthetic 결제 시나리오로 `payment -> producer -> ticket consumer` 연결이 기대대로 보이는지 먼저 검증한다.
5. 파일럿 결과가 안정적이면 reservation-service와 ticket-service producer 호출부를 같은 wrapper 방식으로 일괄 교체한다.
6. 전체 교체 후 raw `send_and_wait()` 직접 호출 검색 테스트를 추가한다.
7. Tempo에서 `payment -> producer -> ticket consumer -> ticket producer -> notification consumer` 연결을 확인한다.

## 닫기 기준

- 서비스 코드에서 raw Kafka producer 발행 조립이 사라진다.
- producer wrapper의 `send_and_wait()`만 사용해도 producer span과 Kafka trace header가 남는다.
- payment outbox dispatcher는 저장된 `trace_context`를 wrapper option으로 넘긴다.
- producer span이 원 요청 trace id 아래에 붙는 것을 테스트와 Tempo 화면으로 확인한다.
- 공식 aiokafka instrumentation 사용 여부를 결정하고, 사용할 경우 중복 span이 없는지 검증한다.
