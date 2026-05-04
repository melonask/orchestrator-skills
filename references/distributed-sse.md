# Distributed SSE (Redis Pub/Sub to Axum)

In a microservice architecture, the container executing a background task (e.g., an Apalis Worker processing an image) is almost never the same container holding the user's open HTTP request.

To stream progress events to the user, the **Worker** publishes messages to Redis, and the **API** subscribes to Redis and translates those messages into HTTP Server-Sent Events (SSE).

## Dependencies

```toml
[dependencies]
redis = { version = "1.2.1", features = ["tokio-comp"] }
axum = "0.8.9"
tokio-stream = "0.1.18"
futures-util = "0.3.32"
async-stream = "0.3.6"
```

## The Pattern

### 1. Worker (Publisher)

When the background worker completes a step, it publishes a JSON payload to a Redis channel specific to the Task ID. Multiplexed Redis connections are perfectly safe to share for `PUBLISH` operations.

```rust
use redis::AsyncCommands;

async fn worker_process(redis_url: &str, task_id: &str) {
    let client = redis::Client::open(redis_url).unwrap();
    let mut conn = client.get_multiplexed_tokio_connection().await.unwrap();

    let channel = format!("evt.task.{}", task_id);
    let payload = r#"{"progress": 50, "status": "running"}"#;

    // Publish the event to the channel
    let _: () = conn.publish(channel, payload).await.unwrap();
}
```

### 2. API (Subscriber & SSE Stream)

The Axum handler MUST acquire a **dedicated Pub/Sub connection** (a shared multiplexed connection will panic if you attempt to call `SUBSCRIBE` on it). It wraps the Redis stream into an `axum::response::sse::Event` and yields it.

```rust
use axum::{extract::Path, response::sse::{Event, Sse, KeepAlive}};
use futures_util::StreamExt;
use std::convert::Infallible;
use std::time::Duration;

async fn task_events_handler(Path(task_id): Path<String>)
    -> Sse<impl futures_util::stream::Stream<Item = Result<Event, Infallible>>>
{
    let client = redis::Client::open("redis://127.0.0.1/").unwrap();

    let stream = async_stream::stream! {
        // 1. Get a dedicated connection for Pub/Sub
        let mut pubsub = client.get_async_pubsub().await.unwrap();

        // 2. Subscribe to the specific task's channel
        let channel = format!("evt.task.{}", task_id);
        pubsub.subscribe(&channel).await.unwrap();

        let mut msg_stream = pubsub.on_message();

        // 3. Yield events as they arrive from Redis
        while let Some(msg) = msg_stream.next().await {
            let payload: String = msg.get_payload().unwrap();

            yield Ok(Event::default().data(payload.clone()));

            // Optional: Break the stream if the task reports completion
            if payload.contains(r#""status":"done""#) {
                break;
            }
        }
    };

    // Keep the HTTP connection alive with pings every 15 seconds
    Sse::new(stream).keep_alive(
        KeepAlive::new()
            .interval(Duration::from_secs(15))
            .text("ping")
    )
}
```
