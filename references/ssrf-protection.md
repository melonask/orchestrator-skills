# SSRF Protection - External RPC Clients

External RPC clients must reject loopback, link-local, private, and otherwise non-public IP targets unless the target is explicitly allow-listed. Do this at the connector/resolver boundary so redirects and DNS answers are checked immediately before TCP connect.

## Dependencies

```toml
[dependencies]
reqwest = { version = "0.13.3", default-features = false, features = ["rustls", "json"] }
tokio = { version = "1.52.1", features = ["net"] }
ipnet = "2.12.0"
```

For lower-level Hyper clients, current Hyper uses `hyper-util`:

```toml
[dependencies]
hyper-util = { version = "0.1.20", features = ["client", "client-legacy", "tokio"] }
tower = "0.5.3"
ipnet = "2.12.0"
```

## Pattern 1: Reqwest DNS Resolver Guard

Use `reqwest::ClientBuilder::dns_resolver` for current `reqwest`. Do not hand-roll URL string checks; they miss redirects, alternate DNS answers, IPv6, and direct IP hosts.

```rust
use reqwest::dns::{Addrs, Name, Resolve, Resolving};
use ipnet::IpNet;
use std::{collections::HashSet, net::{IpAddr, SocketAddr}, sync::LazyLock};

static BLOCKED_NETS: LazyLock<Vec<IpNet>> = LazyLock::new(|| {
    [
        "0.0.0.0/8", "10.0.0.0/8", "100.64.0.0/10", "127.0.0.0/8",
        "169.254.0.0/16", "172.16.0.0/12", "192.0.0.0/24", "192.0.2.0/24",
        "192.168.0.0/16", "198.18.0.0/15", "198.51.100.0/24", "203.0.113.0/24",
        "224.0.0.0/4", "240.0.0.0/4", "::/128", "::1/128", "::ffff:0:0/96",
        "64:ff9b:1::/48", "100::/64", "2001:db8::/32", "fc00::/7", "fe80::/10",
        "ff00::/8",
    ]
    .into_iter()
    .map(|cidr| cidr.parse().expect("valid blocked CIDR"))
    .collect()
});

#[derive(Clone, Default)]
pub struct SsrfSafeResolver {
    allow_list: HashSet<IpAddr>,
}

impl SsrfSafeResolver {
    pub fn new(allow_list: impl IntoIterator<Item = IpAddr>) -> Self {
        Self { allow_list: allow_list.into_iter().collect() }
    }
}

impl Resolve for SsrfSafeResolver {
    fn resolve(&self, name: Name) -> Resolving {
        let host = name.as_str().to_owned();
        let allow_list = self.allow_list.clone();

        Box::pin(async move {
            let addrs: Vec<SocketAddr> = tokio::net::lookup_host((host.as_str(), 0))
                .await?
                .filter(|addr| is_allowed_external_ip(addr.ip(), &allow_list))
                .collect();

            if addrs.is_empty() {
                return Err(std::io::Error::new(
                    std::io::ErrorKind::PermissionDenied,
                    "blocked by SSRF IP policy",
                ).into());
            }

            Ok(Box::new(addrs.into_iter()) as Addrs)
        })
    }
}

fn is_allowed_external_ip(ip: IpAddr, allow_list: &HashSet<IpAddr>) -> bool {
    if allow_list.contains(&ip) {
        return true;
    }

    !BLOCKED_NETS.iter().any(|net| net.contains(&ip))
}

pub fn external_rpc_client(allow_list: HashSet<IpAddr>) -> reqwest::Result<reqwest::Client> {
    reqwest::Client::builder()
        .https_only(true)
        .no_proxy()
        .dns_resolver(SsrfSafeResolver::new(allow_list))
        .build()
}
```

## Pattern 2: Hyper HttpConnector Guard

If the codebase uses Hyper directly, wrap the resolver passed to `hyper_util::client::legacy::connect::HttpConnector::new_with_resolver`. The old `hyper::client::HttpConnector` path is stale for current Hyper. This snippet uses the same `is_allowed_external_ip` helper shown above.

```rust
use hyper_util::client::legacy::connect::{dns::Name, dns::GaiResolver, HttpConnector};
use std::{collections::HashSet, future::Future, net::{IpAddr, SocketAddr}, task::{Context, Poll}};
use tower::Service;

#[derive(Clone)]
pub struct FilteringResolver {
    inner: GaiResolver,
    allow_list: HashSet<IpAddr>,
}

impl Service<Name> for FilteringResolver {
    type Response = std::vec::IntoIter<SocketAddr>;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = std::pin::Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx).map_err(Into::into)
    }

    fn call(&mut self, name: Name) -> Self::Future {
        let mut inner = self.inner.clone();
        let allow_list = self.allow_list.clone();

        Box::pin(async move {
            let addrs = inner.call(name).await?;
            let filtered: Vec<_> = addrs
                .filter(|addr| is_allowed_external_ip(addr.ip(), &allow_list))
                .collect();

            if filtered.is_empty() {
                return Err(std::io::Error::new(
                    std::io::ErrorKind::PermissionDenied,
                    "blocked by SSRF IP policy",
                ).into());
            }

            Ok(filtered.into_iter())
        })
    }
}

pub fn guarded_http_connector(allow_list: HashSet<IpAddr>) -> HttpConnector<FilteringResolver> {
    let resolver = FilteringResolver { inner: GaiResolver::new(), allow_list };
    let mut connector = HttpConnector::new_with_resolver(resolver);
    connector.enforce_http(false); // allow HTTPS schemes to pass to the TLS connector layer
    connector
}
```

## Rules

- Always filter resolved `SocketAddr` values, not just request host strings.
- Disable ambient proxies with `no_proxy()` for external RPC clients unless proxy egress is separately controlled.
- Keep allow-lists explicit and narrow; never allow-list private ranges wholesale.
- Re-run this check on every DNS resolution so DNS rebinding cannot reuse a stale approval.
