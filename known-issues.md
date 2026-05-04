# Known Issues — Orchestrator Skill

Issues discovered through practical compilation testing against actual crate versions (May 2026).

## 1. `hmac` v0.13 requires `KeyInit` trait import (FIXED)

The `Hmac::<Sha256>::new_from_slice()` call fails without importing `KeyInit`:

```rust
// WRONG:
use hmac::{Hmac, Mac};
let mut mac = Hmac::<Sha256>::new_from_slice(key)?;

// CORRECT:
use hmac::{Hmac, Mac, KeyInit};
let mut mac = Hmac::<Sha256>::new_from_slice(key)?;
```

The `KeyInit` trait must be in scope for `new_from_slice` to be available on `Hmac<D>`.

## 2. `argon2` v0.6.0-rc.8 API Changes (FIXED)

### `hash_password` signature changed
`hash_password` no longer takes a separate salt parameter. Salt is now generated independently
and embedded in the hash string:

```rust
// WRONG (skill's old code):
argon2.hash_password(secret.as_bytes(), &salt)?

// CORRECT (argon2 0.6.x):
argon2.hash_password(secret.as_bytes())?
```

### `rand_core` not re-exported from `argon2::password_hash`
Use `rand_core` directly as a dependency (v0.9):

```rust
// WRONG:
use argon2::password_hash::rand_core::OsRng;

// CORRECT:
use rand_core::OsRng;
```

### `verify_password` takes `&str` not parsed `PasswordHash`
```rust
// WRONG:
argon2.verify_password(secret.as_bytes(), &parsed_hash)

// CORRECT:
argon2.verify_password(secret.as_bytes(), stored_hash_str)
```

### Updated dependency:
```toml
argon2 = "0.6.0-rc.8"
rand_core = "0.9"  # required for OsRng
```

## 3. `hyper-util` `dns` Feature

The `hyper-util` crate does NOT have a `dns` feature flag. The SSRF protection pattern
described in the skill that uses `hyper-util` with `dns` feature cannot work as documented.
Use `hickory-dns` or `trust-dns-resolver` instead for custom DNS resolution.
