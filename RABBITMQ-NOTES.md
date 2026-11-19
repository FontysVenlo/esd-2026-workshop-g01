# Event-driven architecture and RabbitMQ — workshop notes

These are informal notes and ideas for a one-hour ish workshop. The main idea is **event-driven architecture**; RabbitMQ is the tool we use to see it in action.

## Start with a familiar situation

An online shop receives an order. Several things need to happen: send a confirmation email, update inventory, and maybe award loyalty points.

- **Synchronous communication:** the Order Service calls another service and waits for its response. This is useful when it needs an immediate answer, such as whether a payment was approved. If the other service is slow or unavailable, the caller may have to wait or handle a failure.
- **Asynchronous communication:** the Order Service publishes a message and continues without waiting for each interested service to finish. Other services process the message when they can. The trade-off is that results may not be immediate, and failures still need handling.

**Ask students:** Does the customer need to wait for the confirmation email to be sent before seeing “order received”?

## What is event-driven architecture?

In event-driven architecture, parts of a system communicate by producing and reacting to **events**. An event describes something that has happened, such as `OrderPlaced`. Interested services can react independently.

```text
Order Service ──publishes OrderPlaced──> RabbitMQ
                                          ├──> notifications queue ──> Email Service
                                          └──> inventory queue ──────> Inventory Service
```

The Order Service does not need to call each service directly. A new Loyalty Service could receive the same kind of event through its own queue. This reduces direct dependencies, but it also makes the overall flow less immediate and sometimes harder to trace or debug.

An event is different from a command: `OrderPlaced` says what happened; `SendEmail` asks someone to do something. Both can be sent as messages, but they express different intentions.

## What is RabbitMQ?

RabbitMQ is a **message broker**: software that receives messages from applications and routes them to queues, where consumers can receive them. Think of it as a delivery hub between services, not the service that actually sends emails or updates stock.

- **Producer:** an application that publishes a message. Here, the Order Service is the producer.
- **Message:** the data sent, often JSON, plus metadata. An event is the meaning carried by a message.
- **Exchange:** receives published messages and decides where to route them.
- **Queue:** holds messages until they are delivered to consumers or otherwise removed.
- **Binding:** a rule connecting an exchange to a queue.
- **Consumer:** an application that receives and processes messages from a queue.

Example message:

```json
{
  "eventType": "OrderPlaced",
  "orderId": "ORD-001",
  "product": "Workshop mug",
  "quantity": 2
}
```

The repository's [practical exercise](README.md) uses `OrderPlaced`; `OrderCreated` would be another reasonable name for the same teaching example.

## How does a message get to the right service?

1. The Order Service publishes an event to an **exchange**.
2. The exchange checks its **bindings** and routes the message to matching queues.
3. Each service consumes from its own queue and performs its own work.

Some common exchange types:

- **Fanout:** sends a copy to every bound queue. Good for our first demonstration: email and inventory both react to an order.
- **Direct:** routes using an exact routing-key match, for example `order.placed`.
- **Topic:** routes using routing-key patterns, for example `order.*`.

A routing key is a label supplied when publishing; a binding can use it to decide what a queue receives. Fanout ignores the routing key. If no queue matches, a message may be discarded unless the publisher has configured a way to detect or handle that situation.

**Important distinction:** two separate queues can each receive a copy of an event. Two consumers sharing **one** queue normally compete for messages: each message is delivered to one of them, rather than to both. This is useful for sharing a workload.

## What if a service is unavailable?

If the Inventory Service is offline but its queue already exists and is bound, RabbitMQ can keep routed messages in that queue. When the service reconnects, it can consume the waiting messages. The Email Service can continue independently.

This is **buffering**, not unlimited storage or a guarantee that nothing can ever go wrong. Queue limits, message expiry, broker failure, and configuration all matter. A queue created only after an event was published does not automatically receive that earlier event.

## Basic reliability ideas

- **Acknowledgement (ack):** after successful processing, the consumer tells RabbitMQ it handled the message. If a consumer disconnects before acknowledging, RabbitMQ can requeue the unacknowledged message for delivery again.
- **Durable queue and persistent message:** help messages survive a broker restart when configured appropriately. They are useful pieces of reliability, not a complete guarantee by themselves.
- **Publisher confirm:** RabbitMQ tells the producer whether it accepted responsibility for a published message. The producer still needs a strategy for failures and unroutable messages.
- **Idempotency:** processing the same event more than once should not cause an unwanted repeated effect. For example, record processed event IDs before reducing stock again. Redelivery can happen, so do not assume exactly-once business processing.

A practical example: if inventory is updated but the consumer crashes before sending its ack, the message may be delivered again. How can the service avoid subtracting the same stock twice?

## Possible one-hour flow

| Time | Activity |
| --- | --- |
| 0–10 min | Discuss the shop example and synchronous vs asynchronous communication. |
| 10–20 min | Introduce events, producer, exchange, binding, queue and consumer; draw the flow together. |
| 20–40 min | Follow the [dashboard exercise](README.md): publish one order and observe two queues. |
| 40–50 min | Discuss an offline consumer, acknowledgements and duplicate processing. |
| 50–60 min | Students propose a third service, add a queue and binding, and predict what happens when another order is published. |

Setup needs Docker Desktop and the repository's Compose configuration. Follow the README to start RabbitMQ and open its dashboard; no prior RabbitMQ installation is assumed. Allow extra time before the workshop for the first image download.

## Questions and ideas to explore

- Which parts of the checkout need an immediate response, and which can happen later?
- What changes if email and inventory share one queue instead of having separate queues?
- What happens if inventory is offline for an hour? What if it never comes back?
- Add a Loyalty Service: what should it do when it receives an order event?
- How would you investigate an order whose email never arrived?

## Further reading

- [RabbitMQ tutorials: publish/subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-javascript)
- [RabbitMQ acknowledgements and publisher confirms](https://www.rabbitmq.com/docs/confirms)
