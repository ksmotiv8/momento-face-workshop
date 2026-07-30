---
name: momento-functions
description: Build serverless compute on Momento - Rust functions compiled to WebAssembly that run next to a Momento cache, with host interfaces for cache, HTTP egress, topics, tokens, Valkey, and AWS. Covers scaffolding, deploying, invoking, the example patterns (proxies, token vending, vector search, event handlers), and debugging memory limits and wasm panics. Use for any task involving Momento Functions or /functions endpoints.
---

# Momento Functions

> **Guard - reference only. Loading this skill is not a request to act.**
> Do not create, build, deploy, or run anything based on this skill unless
> the user has explicitly asked for that work in this conversation. If they
> have not asked, answer from it and stop.

Momento Functions are small WebAssembly programs that run on Momento's
service hosts, colocated with a cache. You write them in Rust, compile to
`wasm32-wasip2`, and upload the wasm to a cache; Momento runs them on
demand. Two properties make them different from a Lambda-style FaaS:

- **Compute next to data.** Cache reads and writes from inside a function
  are local operations, not network hops. Patterns like
  read-model-from-cache, cache-aside for expensive API calls, and
  single-writer state live where the data lives.
- **You are a stateless web server in a sandbox.** Warm instances are
  reused (in-memory caches pay off), but there is no filesystem, no
  stdin/streams, and limited std support. `std::time` works; env vars come
  from function configuration; stdout/stderr go to the system log. Other
  wasi interfaces panic at runtime or fail at upload with a linking error.

Canonical source: https://github.com/momentohq/functions - the crates plus
an `examples/` directory that covers most real patterns (distilled below).

## Two function types

- **Web functions** (`momento-functions-guest-web`): request/response over
  HTTPS. `invoke!(handler)` registers the handler. Invoked by POSTing to
  `https://<endpoint>/functions/<cache>/<name>`.
- **Spawn functions** (`momento-functions-guest-spawn`): fire-and-forget
  work items. `spawn!(handler)`; the handler returns nothing to a caller.

## Scaffold

```bash
cargo init --lib myfn && rustup target add wasm32-wasip2
```

`Cargo.toml`: `crate-type = ["cdylib"]`, then add capability crates as you
need them (each is its own focused crate):
`momento-functions-guest-web` (or `-guest-spawn`), `momento-functions-bytes`,
`momento-functions-cache`, `momento-functions-http`, `momento-functions-topic`,
`momento-functions-token`, `momento-functions-valkey`,
`momento-functions-host-log`, `momento-functions-aws-auth`,
`momento-functions-aws-s3`, `momento-functions-aws-secrets-manager`.

`.cargo/config.toml`: `[build] target = "wasm32-wasip2"`.

Payload in, response out - three styles:

```rust
use momento_functions_bytes::{Data, encoding::Json};
use momento_functions_guest_web::{invoke, WebResult, WebResponse};

invoke!(ping);
fn ping(_payload: Data) -> &'static str { "pong" }          // raw in, str out

fn typed(Json(req): Json<MyReq>) -> WebResult<Json<MyResp>> { … }  // JSON both ways

fn full(payload: Data) -> WebResult<WebResponse> {           // status + headers
    Ok(WebResponse::new().with_status(200)
        .with_headers(vec![("x-thing".into(), "v".into())])
        .with_body("…")?)
}
```

`Data` is a host-side buffer; call `.into_bytes()` only when you need the
bytes in wasm memory. `WebError: From<E: Error>` so `?` works everywhere.

## Request context

`WebEnvironment::load()` exposes the invocation: `cache_name()`,
`function_name()`, `invocation_id()`, `http_method()`, `http_path()`,
`query_parameters()`, `headers()` (match header names case-insensitively),
and `token_metadata()` - the `token_id` payload baked into the calling API
key, which lets you pass caller identity through Momento-signed tokens.
`std::env::vars()` holds env vars from the function's configuration.

## Host interfaces, by example (from `examples/`)

- **Cache ops** (`cache-scalar`): `cache::set(key, bytes, ttl)`,
  `cache::get::<Vec<u8>>(key)`, `cache::delete`, `cache::set_if` with
  conditions (Absent, NotEqual, …) for optimistic concurrency.
- **Logging** (`logging`, `logging-extended`): stdout goes to the system
  log, but the useful pattern publishes logs to a Momento topic you can
  subscribe to live:
  `configure_logs([LogDestination::topic(env.function_name()).into()])?;`
  then use the standard `log::info!` macros. Subscribe with the CLI or SDK
  to tail your function.
- **HTTP egress** (`turbopuffer-*`, `sse-proxy`):
  `momento_functions_http::invoke(Request::new(url, method)
  .with_headers(…).with_body(…))` reaches any external API from inside the
  function.
- **SSE streaming** (`sse-proxy`): functions can return Server-Sent Event
  streams (`sse_streaming_response_with_headers`) and proxy upstream SSE
  APIs, forwarding caller headers. Callers use `curl -N`.
- **Token vending machine** (`token-vending-machine`):
  `generate_disposable_token(Permissions::new().with_cache(…).with_topic(…))`
  issues short-lived, scoped Momento credentials from inside a function -
  an auth service with no Lambda or API Gateway.
- **AWS access** (`s3-get`, `s3-put`, `secrets-manager-get`,
  `dynamodb-accelerator`): federate into an IAM role via
  `momento_functions_aws_auth::provider(&Authorization::Federated(IamRole{role_arn}), region)`,
  then use typed clients (`S3Client`, `SecretsManagerClient`, …) over
  Momento's managed channel. The DynamoDB accelerator example is the
  classic read-through cache in front of a table.
- **Spawn functions** (`spawn-function`, `spawn-function-json`):
  same payload styles, no response - background work triggered by a caller.

## Embeddings and vector search

Functions are a natural home for embedding pipelines because the cache
absorbs the two expensive parts: repeated embedding calls and index
round-trips. Three tiers, all proven in `examples/` or in production
projects built on this skill:

1. **Generate embeddings via an external model** (`fine-foods-embeddings`):
   POST document batches to a function; it calls OpenAI's embeddings API
   over HTTP egress and returns/forwards vectors. Batch on the client
   (~12 MB JSON chunks), map fields with `serde(alias)`.
2. **Cache-aside for query embeddings** (`turbopuffer-search`): before
   calling the embedding API for a query, `cache::get` the query's vector;
   on miss, embed and `cache::set` with a short TTL. Repeat queries skip
   the model entirely.
3. **Index and search**, two backends:
   - *Turbopuffer* (`turbopuffer-index`, `-search`, `-recommend-articles`):
     plain HTTP egress to its REST API; env vars for region/namespace/key.
   - *Momento-managed Valkey* (`valkey-cluster-vector-index`, `-search`):
     `momento_functions_valkey::get_managed_cluster_client` +
     `Command::builder("FT.SEARCH")` with `*=>[KNN {topk} @vector $query_vector]`
     for KNN over HSET-stored vectors; the index example creates the index
     on demand from the first document's dimensionality.

4. **Run the embedding model INSIDE the function** (no external API): bake
   an ONNX model into the wasm with `include_bytes!` and run it with tract
   (`tract-onnx`, pure Rust, compiles to wasm32-wasip2). Proven with a
   13 MB MobileFaceNet/ArcFace face-embedding model: 512-D embeddings,
   L2-normalize so cosine similarity is a dot product, store the reference
   library as a JSON cache key, brute-force match in-function (fine for
   hundreds of entries). Parse the model once behind a `OnceLock` - warm
   instances then embed in ~0.5s. Keep the embedding library in the cache
   rather than the wasm so it updates without redeploys. Critical for
   consistency: use the SAME inference crate and preprocessing on whatever
   side builds the library (a host tool) and the side that queries (the
   function) - embeddings from different runtimes or preprocessing paths
   do not compare reliably.

## Example index

All at https://github.com/momentohq/functions/tree/main/examples - go to
the named directory for full working code.

| Example | What it shows |
|---|---|
| `web-function` | Minimal web function: `invoke!`, raw `Data` in, str out |
| `web-function-json-greeter` | Typed JSON request/response via `Json<T>` |
| `web-function-token-metadata` | Reading caller identity from the API key's `token_id` |
| `reading-and-sending-headers` | Request headers in, custom response headers out |
| `function-environment` | Invocation context: cache/function name, method, path, query params |
| `environment-variables` | Env vars from function configuration |
| `cache-scalar` | Cache get/set/delete/set_if with conditions |
| `logging` | Log to a Momento topic with the standard `log` macros |
| `logging-extended` | Multiple log destinations, per-topic levels, CloudWatch |
| `spawn-function` / `spawn-function-json` | Fire-and-forget functions (no response) |
| `token-vending-machine` | Issue scoped, short-lived Momento tokens from a function |
| `sse-proxy` | Stream Server-Sent Events; proxy an upstream SSE API |
| `s3-get` / `s3-put` | Read/write S3 via federated IAM role |
| `secrets-manager-get` | Fetch AWS Secrets Manager secrets |
| `dynamodb-accelerator` | Read-through cache proxy in front of DynamoDB GetItem |
| `fine-foods-embeddings` | Batch-generate OpenAI embeddings for a dataset |
| `turbopuffer-index` / `-search` | Vector index + KNN search over HTTP, query-embedding cache-aside |
| `turbopuffer-index-articles` / `-search-articles` / `-recommend-articles` | End-to-end article search + recommendations (average embeddings, ANN) |
| `valkey-vector-embeddings` | Embedding producer for the Valkey examples |
| `valkey-cluster-vector-index` / `-search` | FT.CREATE/FT.SEARCH KNN on Momento-managed Valkey |

## Regions and HTTP endpoints

Functions and the HTTP API are addressed through per-region cell endpoints
(full list: https://docs.momentohq.com/platform/regions). **Prefer
us-west-2 unless you have a locality reason not to** - it is the primary
region where new capabilities like Functions land first.

| Region | Code | HTTP API endpoint |
|---|---|---|
| US West (Oregon) - recommended | us-west-2 | `api.cache.cell-4-us-west-2-1.prod.a.momentohq.com` |
| US East (N. Virginia) | us-east-1 | `api.cache.cell-us-east-1-1.prod.a.momentohq.com` |
| US East (Ohio) | us-east-2 | `api.cache.cell-1-us-east-2-1.prod.a.momentohq.com` |
| Europe (Ireland) | eu-west-1 | `api.cache.cell-1-eu-west-1-1.prod.a.momentohq.com` |
| Europe (London) | eu-west-2 | `api.cache.cell-1-eu-west-2-1.prod.a.momentohq.com` |
| Europe (Frankfurt) | eu-central-1 | `api.cache.cell-1-eu-central-1-1.prod.a.momentohq.com` |
| Europe (Stockholm) | eu-north-1 | `api.cache.cell-1-eu-north-1-1.prod.a.momentohq.com` |
| Asia Pacific (Mumbai) | ap-south-1 | `api.cache.cell-1-ap-south-1-1.prod.a.momentohq.com` |
| Asia Pacific (Tokyo) | ap-northeast-1 | `api.cache.cell-ap-northeast-1-1.prod.a.momentohq.com` |
| Asia Pacific (Singapore) | ap-southeast-1 | `api.cache.cell-1-ap-southeast-1-1.prod.a.momentohq.com` |
| Asia Pacific (Sydney) | ap-southeast-2 | `api.cache.cell-1-ap-southeast-2-1.prod.a.momentohq.com` |

Your cache and API key are region-bound: create the cache in the region
whose endpoint you plan to call (enterprise customers can get private
cells in any AWS commercial region). `<endpoint>` in the commands below
means the host from this table.

## Deploy and invoke

```bash
cargo build --release
# Write the deploy body to a FILE. Inlining tens of MB of base64 into the
# curl argv hits the OS ARG_MAX limit ("Argument list too long").
base64 -w0 target/wasm32-wasip2/release/myfn.wasm > /tmp/fn.b64   # Linux
# macOS base64 has no -w; its output is already unwrapped:
#   base64 -i target/wasm32-wasip2/release/myfn.wasm > /tmp/fn.b64
printf '{"inline_wasm":"' > /tmp/fn.json
cat /tmp/fn.b64 >> /tmp/fn.json
printf '"}' >> /tmp/fn.json
curl -X PUT "https://<endpoint>/functions/manage/<cache>/<name>" \
  -H "authorization: $MOMENTO_API_KEY" -H "Content-Type: application/json" \
  --data-binary @/tmp/fn.json                  # expect 204
curl -X POST "https://<endpoint>/functions/<cache>/<name>" \
  -H "authorization: $MOMENTO_API_KEY" --data-binary @payload
```

Environment variables are set in the same manage PUT, under the
`environment_variables` field (verified empirically; the repo README does
not document it):

```json
{"inline_wasm":"<b64>", "environment_variables": {"UPSTREAM_SSE_URL": "https://…"}}
```

Careful: the manage API returns 204 even for unknown payload fields - a
misspelled field (`environment`, `env_vars`) is silently ignored and your
function sees no variables. If `std::env::vars()` comes back empty, check
the field name first.

- Deploy payloads of at least 33 MiB work - embedding a multi-MB ML model
  in the wasm via `include_bytes!` is a viable pattern.
- Invoking `Json<T>` handlers: older runtimes rejected trailing newlines
  (`echo` appends one) with 400 "trailing characters"; current guest
  crates (0.26+) accept them. `printf '%s'` or a file is still the safe
  habit for JSON bodies.
- The edit-build-deploy loop is ~30 seconds; iterate freely.
- DELETE on the manage URL is not supported (405). Re-PUT to replace a
  function; removal goes through the CLI/console.
- CLI alternative (when available): `momento preview function put-function
  / invoke-function`.
- If a transitive dep demands a newer rustc than yours, pin it back.
  Known instance: tract pulls `kstring` 2.0.4 which wants rustc >= 1.96;
  fix with `cargo update kstring@2.0.4 --precise 2.0.2`. General form:
  `cargo update <crate>@<ver> --precise <older-ver>`.

## Operational knowledge (memory, panics, warm state)

- **Memory limits.** Instances have a per-account memory budget (~4 MB by
  default; raisable). The wasm module, `include_bytes!` data, and all
  runtime allocations count. The symptom is 502
  `"Function memory limit exceeded"`, sometimes flaky right at the edge.
  For image work: decoded PIXELS cost the memory, not file bytes (a 60 KB
  JPEG at 1280x720 decodes to ~2.7 MB RGB). Keep inputs small and
  downscale early; pyramid-style CV algorithms multiply the cost.
- **Panics are traps.** A wasm panic surfaces as an opaque 500 "Internal
  server error", while application errors return real messages. With no
  logs configured, debug by payload bisection: garbage input producing a
  clean error proves instantiation and wiring; then vary payload size and
  content to localize. Better: configure topic logging early and tail it.
- **Warm instances.** Cache expensive init (model parsing, compiled plans)
  in a `static OnceLock`. Measured: a 13 MB ONNX parse dropped from 6.5s
  (cold) to 0.5s (warm) per invoke.
- **State patterns.** A function that is the sole writer of a cache key can
  do plain get-modify-set safely (rolling histories: append, trim window,
  cap length, set with TTL). Use `#[serde(default)]` on new fields so old
  cached values keep deserializing. TTL everything: when producers stop,
  the system cleans itself up.
