# orchestrator-skills

A specialized "glue" skill for building dynamic, secure, and distributed Rust microservices. This skill bridges the gap between individual component libraries (Axum, Apalis, SQLx) by providing production-grade integration patterns for distributed architectures.

## Overview

Modern Rust backends—especially those using zero-account designs, dynamic pricing catalogs, and separated API/Worker containers—require specific integration patterns to function securely and reliably.

This skill provides those exact patterns:

- **Crypto Auth**: Fast $O(1)$ database lookups without timing leaks, using HMAC-SHA256 indexes and Argon2id.
- **Dynamic Config**: Parsing complex, nested TOML manifests into application state, and building robust CLI routers for multi-mode execution (Monolith vs. Microservice).
- **Distributed SSE**: Bridging background workers to user-facing HTTP streams using Redis Pub/Sub and Axum.
- **SSRF Protection**: Guarding external reqwest/Hyper RPC clients with connector-level IP filtering.
- **ULID SQLx Mapping**: Using ULID task IDs as 16-byte database values without string storage.
- **S3 Bucket Policies**: Enforcing presigned PUT upload size limits with prefix-scoped bucket policies.

## Installation

```bash
npx skills add melonask/orchestrator-skills
```

## File Structure

```
orchestrator/
├── SKILL.md                          # Core overview and routing
└── references/
    ├── crypto-auth.md                # HMAC-SHA256 indexing, Argon2id, Constant-Time checks
    ├── config-dynamic.md             # TOML parsing, Serde, Clap CLI routing
    ├── distributed-sse.md            # Redis Pub/Sub to Axum Server-Sent Events (SSE)
    ├── ssrf-protection.md            # Reqwest/Hyper external RPC IP filtering
    ├── ulid-sqlx.md                  # ULID to uuid::Uuid and SQLx 16-byte storage
    └── s3-bucket-policy.md           # Presigned PUT size limits via bucket policy
```

## License

Provided as-is for development with LLM assistants.
