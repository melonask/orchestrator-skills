# Crypto Auth — Secure Identifiers & Hashing

In high-performance, passwordless systems (like `reeve`), you often need to verify a secret (like a 20-digit Space ID) quickly while protecting against timing attacks and database leaks.

## Dependencies

```toml
[dependencies]
argon2 = "0.6.0-rc.8" # Password hashing
hmac = "0.13.0"        # Message authentication
sha2 = "0.11.0"        # SHA-256 hash function
subtle = "2.6.1"       # Constant-time comparisons
rand_core = "0.9"      # Cryptographically secure RNG
```

## Pattern 1: Fast Lookup Index (HMAC-SHA256)

Argon2id is too slow to use for searching a database (e.g., `SELECT * FROM users WHERE hash = ?`). Instead, create a fast, deterministic lookup index using an environment-level `pepper`. Truncate the HMAC result to 16 bytes to store it optimally as a standard `BYTEA` / `BLOB` (UUID size).

```rust
use hmac::{Hmac, Mac, KeyInit};
use sha2::Sha256;

/// Create a fast, indexed lookup value (16 bytes)
pub fn create_lookup_index(pepper: &[u8], space_id: &str) -> [u8; 16] {
    let mut mac = Hmac::<Sha256>::new_from_slice(pepper)
        .expect("HMAC can take key of any size");

    mac.update(space_id.as_bytes());
    let result = mac.finalize().into_bytes();

    // Truncate to 16 bytes for optimal DB storage
    let mut index =[0u8; 16];
    index.copy_from_slice(&result[..16]);
    index
}

// SQLx Usage: query!("SELECT hash FROM space WHERE idx = $1", &index)
```

## Pattern 2: Secure Hashing (Argon2id)

Once the row is found via the fast index, verify the actual secret using Argon2id. Configure memory and time costs carefully — API endpoints should use lower memory costs (e.g., 15MB) to prevent memory-exhaustion Denial of Service (DoS) attacks from concurrent requests.

```rust
use argon2::{
    password_hash::{PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2, Params,
};

/// Hash the Space ID for storage
pub fn hash_secret(secret: &str) -> String {
    use rand_core::OsRng;
    let salt = SaltString::generate(&mut OsRng);

    // For API limits: 15MB memory, 2 iterations, 1 parallelism
    let params = Params::new(15360, 2, 1, None).unwrap();
    let argon2 = Argon2::new(argon2::Algorithm::Argon2id, argon2::Version::V0x13, params);

    argon2.hash_password(secret.as_bytes())
        .unwrap()
        .to_string() // Store this string in the database
}

/// Verify the submitted secret against the hash
pub fn verify_secret(secret: &str, stored_hash: &str) -> bool {
    let parsed_hash = match PasswordHash::new(stored_hash) {
        Ok(hash) => hash,
        Err(_) => return false,
    };

    Argon2::default().verify_password(secret.as_bytes(), stored_hash).is_ok()
}
```

## Pattern 3: Constant-Time Comparisons

Whenever comparing sensitive bytes directly (like verifying Webhook payloads or custom signatures), ALWAYS use the `subtle` crate to prevent timing attacks. The standard `==` operator short-circuits and leaks timing data.

```rust
use subtle::ConstantTimeEq;

pub fn is_equal_secure(a: &[u8], b: &[u8]) -> bool {
    if a.len() != b.len() {
        return false; // Length leaks are generally accepted, byte content leaks are not
    }

    // ct_eq() returns a Choice. Unwrap it to a boolean securely.
    a.ct_eq(b).unwrap_u8() == 1
}
```
