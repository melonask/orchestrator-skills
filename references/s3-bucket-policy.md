# S3 Bucket Policies - Presigned PUT Size Limits

Presigned PUT URLs do not carry `content-length-range` conditions. Enforce upload sizes in the object store policy for the exact prefixes that presigned PUTs may write to, and keep the URL signing code focused on method, key, expiry, and required headers.

## Dependencies

```toml
[dependencies]
aws-config = "1.8.16"
aws-sdk-s3 = "1.131.0"
serde_json = "1.0.149"
```

## Pattern

Apply or update a bucket policy with explicit prefix resources and numeric `s3:content-length` denies. This avoids trying to embed POST-only conditions into a PUT presign.

```rust
use aws_sdk_s3::Client;
use serde_json::json;

pub async fn apply_upload_size_policy(
    s3: &Client,
    bucket: &str,
    prefix: &str,
    min_bytes: u64,
    max_bytes: u64,
) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let normalized_prefix = prefix.trim_matches('/');
    let resource = format!("arn:aws:s3:::{bucket}/{normalized_prefix}/*");

    let policy = json!({
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "DenyUploadsBelowMinimumSize",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:PutObject",
                "Resource": resource,
                "Condition": {
                    "NumericLessThan": {
                        "s3:content-length": min_bytes
                    }
                }
            },
            {
                "Sid": "DenyUploadsAboveMaximumSize",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:PutObject",
                "Resource": resource,
                "Condition": {
                    "NumericGreaterThan": {
                        "s3:content-length": max_bytes
                    }
                }
            }
        ]
    });

    s3.put_bucket_policy()
        .bucket(bucket)
        .policy(policy.to_string())
        .send()
        .await?;

    Ok(())
}
```

## Presigned PUT Usage

```rust
use aws_sdk_s3::presigning::PresigningConfig;
use std::time::Duration;

pub async fn presign_upload(
    s3: &aws_sdk_s3::Client,
    bucket: &str,
    key: &str,
    content_type: &str,
) -> Result<String, Box<dyn std::error::Error + Send + Sync>> {
    let request = s3
        .put_object()
        .bucket(bucket)
        .key(key)
        .content_type(content_type)
        .presigned(PresigningConfig::expires_in(Duration::from_secs(900))?)
        .await?;

    Ok(request.uri().to_string())
}
```

## Rules

- Use bucket policy for PUT size enforcement; `content-length-range` is a presigned POST policy condition, not a PUT URL parameter.
- Scope each policy statement to the upload prefix, for example `incoming/tasks/*`, not the whole bucket.
- Merge these statements into any existing bucket policy; `put_bucket_policy` replaces the full policy document.
- Keep application-side validation too; bucket policy is the final server-side guardrail.
- Confirm the deployed S3-compatible server supports `s3:content-length` policy conditions before relying on this for enforcement.
