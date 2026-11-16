# CICD2 Lab 7 — RabbitMQ Fundamentals

## Try it yourself — `delivery_method`

The order model now includes `delivery_method` (for example `delivery` or `collection`). FastAPI publishes it in the queued JSON body, and the worker prints it with the order details, for example:

`Processed: 2 x Phone (delivery)`

## Observations

### What is the difference between a producer, a queue and a consumer?

The **producer** (FastAPI) creates and publishes messages. The **queue** (`orders_queue` in RabbitMQ) stores those messages until they are delivered. The **consumer** (`worker.py`) receives messages from the queue, processes them, and acknowledges them.

### When the worker was stopped, why did the Ready count increase?

With no consumer connected, published messages stay in the queue waiting for delivery. Each new order therefore increases the Ready count because nothing is taking messages out of the queue.

### What changed in RabbitMQ Management while the worker was processing a message?

Consumers became 1. One message moved from Ready to Unacknowledged while the worker slept for 12 seconds. After the worker finished and ACKed, Unacknowledged returned to 0 and Ready decreased.

### What does HTTP 202 Accepted tell the client, and what does it not guarantee?

HTTP 202 means the request was accepted for later processing and the order was queued. It does not guarantee that a worker has processed the order, completed payment, updated a database, or finished any business logic.

### Why is `await` useful when a FastAPI endpoint communicates with RabbitMQ?

Connecting to RabbitMQ and publishing a message are network I/O. `await` pauses that request without blocking the event-loop thread, so Uvicorn can keep handling other requests while waiting for RabbitMQ to respond.

### What changed when you set `prefetch_count=2`? Did the worker actually process two orders at the same time?

RabbitMQ could deliver up to two unacknowledged messages to the worker, so Unacknowledged could reach 2. The worker did **not** process two orders concurrently: the `async for` loop still handled messages one after another.

### With two consumers using `prefetch_count=1`, what did the Consumers and Unacknowledged counts show?

Consumers showed 2. Unacknowledged could reach 2 because each worker may hold one unacknowledged message at a time. RabbitMQ distributed delivery attempts between the two competing workers.

### What happened when one worker stopped before acknowledging an order? Why can the same message be delivered twice?

When Worker A was stopped mid-processing, its unacknowledged message was requeued and could be delivered again to Worker B. RabbitMQ tracks delivery and acknowledgements, not whether the business work already happened, so a retried message must be safe to handle again (idempotent).

### In your own words, what does an event loop do while an async function is awaiting a network operation?

While a coroutine is waiting on network I/O, the event loop can run other ready work. When the network result arrives, the loop resumes the paused coroutine. That gives concurrency during waiting without dedicating a thread to every request.
