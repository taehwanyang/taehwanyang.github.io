---
title: "Event-driven 아키텍처로 구현한 DDD 모듈러 모놀리스 애플리케이션"
date: 2026-04-08
---

## 프로젝트 소개
  - 자바 스프링부트로 간단한 Event-driven 아키텍처를 구현한 애플리케이션입니다.
  - DDD로 모듈화한 모듈러 모놀리스로 바운디드 컨텍스트 간 상태 변경은 Event 기반입니다.
  - 첫번째로, Payment가 완료된 후 이벤트를 발행하면 Order의 상태가 변경됩니다.
    - ApplicationEventPublisher와 TransactionalEventListner로 간단하게 구현합니다.
    - 이벤트가 메모리에만 있으므로 갑자기 서버가 죽으면 이벤트는 처리 도중 사라집니다. 서버가 재가동했을 때 이벤트의 처리 상태를 알 수 없습니다.
  - 두번째는, Order의 상태가 변경됐을 때 Delivery를 생성하고 Notification을 전송합니다. 
    - outbox 패턴을 사용하여 이벤트를 polling 합니다.
    - 이벤트의 상태를 저장하므로 갑자기 서버가 죽어도 이후에 이벤트의 상태를 조회해서 알맞게 처리할 수 있습니다.

## 깃헙 레포지토리
  - [DDD로 구현된 모듈러 모놀리스 Ecommerce Example](https://github.com/taehwanyang/ecommerce-example)

## 개발 스택
  - Java : Spring Boot

## 구현 상세
### Payment
  - PaymentService에서 Payment를 가져온 후 상태를 변경한다. (READY -> APPROVED)
  - ApplicationEventPublisher의 publishEvent로 이벤트 발행

```java
@Service
public class PaymentService {

    private final PaymentRepository paymentRepository;
    private final ApplicationEventPublisher applicationEventPublisher;

    // ...

    @Transactional
    public void approvePayment(Long paymentId) {
        Payment payment = paymentRepository.findById(paymentId)
                .orElseThrow(() -> new IllegalArgumentException("Payment does not exist. paymentId=" + paymentId));

        payment.approve();

        PaymentCompletedEvent event = new PaymentCompletedEvent(
                UUID.randomUUID().toString(),
                payment.getId(),
                payment.getOrderId(),
                payment.getUserId(),
                payment.getAmount()
        );

        applicationEventPublisher.publishEvent(event);
    }
}
```

### Order
  - @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    - Payment의 상태 변경이 데이터베이스에 반영되면 실행되고 롤백됐다면 실행되지 않는다.
  - @Transactional(propagation = Propagation.REQUIRES_NEW)
    - Order의 상태를 변경해야 하므로 새로운 트랜잭션을 만든다.
  - @Async
    - 다른 스레드에서 실행된다.
    - 이후에 kafka를 이벤트 허브로 사용해 마이크로서비스로 전환할 수 있다.
  - Outbox 패턴
    - Transactional Outbox Pattern
    - Distributed System, Microservices Architecture, Event-driven architecture에 쓰인다.
    - DB 트랜잭션과 메시지 발행을 하나로 묶는 역할을 한다.

```java
@Component
public class OrderPaymentEventHandler {
    // ...
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void handle(PaymentCompletedEvent event) {
        try {
            Order order = orderRepository.findById(event.orderId())
                    .orElseThrow(() -> new IllegalArgumentException(
                            "Order does not exist. orderId=" + event.orderId()
                    ));

            order.markPaid();

            OrderPaidEvent orderPaidEvent = new OrderPaidEvent(
                    UUID.randomUUID().toString(),
                    order.getId(),
                    order.getUserId()
            );

            String payload = objectMapper.writeValueAsString(orderPaidEvent);

            // OutboxEvent를 만든 후
            OutboxEvent outboxEvent = OutboxEvent.pending(
                    orderPaidEvent.eventId(),
                    "ORDER",
                    String.valueOf(order.getId()),
                    "OrderPaidEvent",
                    payload
            );
            // 데이터베이스에 저장한다.
            // 이후 이벤트를 모든 consumer가 소비하면 이벤트의 상태를 변경한다.(PENDING -> SENT)
            outboxRepository.save(outboxEvent);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("OrderPaidEvent serialization failed", e);
        } catch (Exception e) {
            log.error("Failed to handle PaymentCompletedEvent. paymentId={}, orderId={}",
                    event.paymentId(), event.orderId(), e);
            throw e;
        }
    }
}
```

### Outbox
  - OutboxPublisher
    - 주기적으로 polling하면서 데이터베이스에서 PENDING 상태인 이벤트를 가져와서 OutboxEventDispatcher에 전달해 실행하도록 위임한다.

```java
@Component
public class OutboxPublisher {
    // ...

    // 3000ms 주기
    // publish는 @Transactional이 아니다.
    // PENDING 상태의 이벤트 목록을 가져와 각 이벤트를 트랜잭션 처리 함수롤 전달한다.
    @Scheduled(fixedDelay = 3000)
    public void publish() {
        List<OutboxEvent> events =
                outboxRepository.findTop100ByStatusOrderByIdAsc(OutboxStatus.PENDING);

        for (OutboxEvent event : events) {
            publishSingleEvent(event.getId());
        }
    }

    // Event를 하나씩 가져와 OutboxEventDispatcher에 이벤트 타입에 따른 처리를 위임한다.
    @Transactional
    public void publishSingleEvent(Long outboxEventId) {
        OutboxEvent event = outboxRepository.findById(outboxEventId)
                .orElseThrow(() -> new IllegalArgumentException(
                        "Outbox event does not exist. outboxEventId=" + outboxEventId
                ));

        if (event.getStatus() != OutboxStatus.PENDING) {
            log.info("Skipping outbox event because status is not PENDING. eventId={}, status={}",
                    event.getEventId(),
                    event.getStatus()
            );
            return;
        }

        try {
            log.info("Publishing outbox event. eventId={}, eventType={}, aggregateType={}, aggregateId={}",
                    event.getEventId(),
                    event.getEventType(),
                    event.getAggregateType(),
                    event.getAggregateId()
            );

            outboxEventDispatcher.dispatch(event);
            event.markSent();

            log.info("Outbox event published successfully. eventId={}", event.getEventId());
        } catch (Exception e) {
            log.error("Failed to publish outbox event. eventId={}", event.getEventId(), e);
            event.markFailed();
        }
    }
}
```

  - OutboxEventDispatcher
    - 이벤트 타입에 따라 handler를 호출해서 이벤트를 다른 바운디드 컨텍스트에 전달 및 처리한다.

```java
@Component
public class OutboxEventDispatcher {
    // ...

    // 이벤트 타입에 따라 이벤트를 처리하는 핸들러로 분기 처리한다.
    @Transactional
    public void dispatch(OutboxEvent outboxEvent) {
        if ("OrderPaidEvent".equals(outboxEvent.getEventType())) {
            handleOrderPaidEvent(outboxEvent);
            return;
        }

        throw new IllegalArgumentException("Unsupported event type. eventType=" + outboxEvent.getEventType());
    }

    // OrderPaidEvent 이벤트 핸들러 함수
    private void handleOrderPaidEvent(OutboxEvent outboxEvent) {
        try {
            OrderPaidEvent event = objectMapper.readValue(
                    outboxEvent.getPayload(),
                    OrderPaidEvent.class
            );

            deliveryService.createDelivery(event);
            notificationService.sendOrderPaidNotification(event);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("OrderPaidEvent deserialization failed", e);
        }
    }
}
```