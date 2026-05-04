---
name: orchestrator
description: "Guide for integrating distributed Rust microservices. Use this skill whenever the user wants to: implement secure passwordless authentication (Argon2id, HMAC-SHA256, subtle constant-time comparisons); create fast $O(1)$ database lookup indexes for secrets; parse complex dynamic TOML configuration files into application state using Serde; build operational CLI applications with Clap subcommands; implement distributed Server-Sent Events (SSE); protect reqwest or Hyper external RPC clients against SSRF; map ULID task IDs through uuid and SQLx as 16-byte database values; or enforce S3-compatible presigned PUT upload limits via bucket policies. Make sure to use this skill when the user mentions Redis Pub/Sub to SSE, TOML dynamic config, Clap CLI, Argon2id hashing, HMAC lookup index, reqwest SSRF, private IP filtering, ULID SQLx mapping, presigned PUT, bucket policy, or microservice glue patterns."
---

# Orchestrator — Microservice Integration Patterns

Building a distributed Rust backend requires specialized "glue" code to connect web frameworks, background workers, and databases securely. This skill provides the architectural patterns needed to stitch these components together.

## Quick Reference: Which Guide Do I Need?

- **Fast secret lookups, Argon2id hashing, timing-attack prevention** -> Read `references/crypto-auth.md`
- **TOML catalog parsing, Serde, Clap CLI subcommands** -> Read `references/config-dynamic.md`
- **Streaming events from Workers to API via Redis Pub/Sub** -> Read `references/distributed-sse.md`
- **External RPC SSRF protection for reqwest/Hyper** -> Read `references/ssrf-protection.md`
- **ULID task IDs stored as 16-byte SQL values** -> Read `references/ulid-sqlx.md`
- **Presigned PUT size limits via S3 bucket policy** -> Read `references/s3-bucket-policy.md`

## Core Patterns at a Glance

### 1. Secure Crypto Auth & Lookups

When dealing with secret identifiers (like a 20-digit Space ID), never store them in plaintext, and never use slow hashes (like Argon2id) for database `SELECT` lookups. Instead, use a two-step approach:

1. **HMAC-SHA256 Fast Index**: Hash the ID with a server-side pepper, truncate to 16 bytes, and use this for the database lookup (`WHERE idx = $1`).
2. **Argon2id Verification**: Once the row is found, perform a constant-time verification against the stored Argon2id hash.

### 2. Dynamic Configurations & CLI

Avoid hard-coded endpoints and prices. Parse a `reeve.toml` catalog into memory at startup using `serde` and `toml`. Use `clap` subcommands to allow a single binary to act as an API server, a background worker, or a migration tool based on the arguments provided.

### 3. Distributed Server-Sent Events (SSE)

In a microservice topology, the API container holding the user's HTTP request is not the same container executing the background task.

- **Worker**: Publishes progress to a Redis channel (`task:<id>:events`).
- **API**: Acquires a dedicated `redis::aio::PubSub` connection, subscribes to the channel, and yields `axum::response::sse::Event` items back to the user via `async_stream`.

### 4. External RPC SSRF Protection

External `reqwest` clients must reject private, loopback, link-local, and unspecified IPs after DNS resolution unless an IP is explicitly allow-listed. Use the current `reqwest::ClientBuilder::dns_resolver` or a guarded `hyper_util::client::legacy::connect::HttpConnector` resolver; do not rely on URL string checks.

### 5. ULID Task IDs and SQLx

Use `ulid::Ulid`, convert to `uuid::Uuid` with `ulid.into()`, and store the 16-byte representation. For `BYTEA`, bind `uuid.as_bytes().as_slice()`; for SQL `UUID`, enable SQLx's `uuid` feature and bind `Uuid` directly.

### 6. S3 Presigned PUT Size Enforcement

Do not try to add `content-length-range` to presigned PUT URLs. Apply an S3-compatible bucket policy with `s3:content-length` numeric deny rules scoped to the upload prefixes.

## Dependency Matrix (Refresh Before Use)

| Feature         | Crates Required                                                                 |
| --------------- | ------------------------------------------------------------------------------- |
| **Crypto Auth** | `argon2 0.6.0-rc.8`, `hmac 0.13.0`, `sha2 0.11.0`, `subtle 2.6.1`, `rand_core 0.10.1` |
| **Config/CLI**  | `toml 1.1.2+spec-1.1.0`, `serde 1.0.228`, `clap 4.6.1`                         |
| **Pub/Sub SSE** | `redis 1.2.1`, `axum 0.8.9`, `tokio-stream 0.1.18`, `async-stream 0.3.6`, `futures-util 0.3.32` |
| **SSRF Guard**  | `reqwest 0.13.3`, `tokio 1.52.1`, `hyper-util 0.1.20`, `tower 0.5.3`, `ipnet 2.12.0` |
| **ULID SQLx**   | `ulid 1.2.1`, `uuid 1.23.1`, `sqlx 0.9.0-alpha.1`                               |
| **S3 Policy**   | `aws-sdk-s3 1.131.0`, `aws-config 1.8.16`, `serde_json 1.0.149`                 |
