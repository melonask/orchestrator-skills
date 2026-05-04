# ULID & SQLx - 16-Byte Task IDs

Task IDs use ULID for sortable, random identifiers. Do not store ULIDs as strings unless the schema explicitly requires human-readable IDs; store the 16 raw bytes.

## Dependencies

```toml
[dependencies]
ulid = { version = "1.2.1", features = ["uuid"] }
uuid = "1.23.1"
sqlx = { version = "0.9.0-alpha.1", features = ["postgres", "runtime-tokio", "uuid"] }
```

## Pattern

Use `ulid::Ulid`, convert to `uuid::Uuid` with `into()`, then bind the 16 bytes for a `BYTEA` column. If the database column type is `UUID` instead of `BYTEA`, bind the `Uuid` directly and let SQLx's `uuid` feature encode it.

```rust
use sqlx::Row;
use ulid::Ulid;
use uuid::Uuid;

pub struct TaskRow {
    pub id: Uuid,
    pub status: String,
}

pub fn new_task_id() -> Uuid {
    let ulid = Ulid::new();
    ulid.into()
}

pub async fn insert_task(pool: &sqlx::PgPool) -> Result<Uuid, sqlx::Error> {
    let task_id = new_task_id();

    sqlx::query(r#"INSERT INTO tasks (id) VALUES ($1)"#)
        .bind(task_id.as_bytes().as_slice()) // BYTEA, exactly 16 bytes
        .execute(pool)
        .await?;

    Ok(task_id)
}

pub async fn load_task(pool: &sqlx::PgPool, task_id: Uuid) -> Result<Option<TaskRow>, sqlx::Error> {
    let Some(row) = sqlx::query(r#"SELECT id, status FROM tasks WHERE id = $1"#)
        .bind(task_id.as_bytes().as_slice())
        .fetch_optional(pool)
        .await?
    else {
        return Ok(None);
    };

    let id_bytes: Vec<u8> = row.try_get("id")?;
    let id = Uuid::from_slice(&id_bytes).map_err(|error| sqlx::Error::Decode(Box::new(error)))?;
    let status = row.try_get("status")?;

    Ok(Some(TaskRow { id, status }))
}
```

For a PostgreSQL `UUID` column, bind the `Uuid` value instead:

```rust
sqlx::query(r#"INSERT INTO tasks (id) VALUES ($1)"#)
    .bind(task_id)
    .execute(pool)
    .await?;
```

## Rules

- Generate IDs with `Ulid::new()`; do not use random strings or database sequences for task IDs.
- Convert with `let id: uuid::Uuid = ulid.into();`; do not parse through a string.
- For `BYTEA`, bind `id.as_bytes().as_slice()` and keep a `CHECK (octet_length(id) = 16)` constraint.
- For `UUID`, enable SQLx's `uuid` feature and bind `Uuid` directly.
- Use `sqlx::query` in reusable snippets; `query!` requires a database connection or prepared offline metadata at compile time.
