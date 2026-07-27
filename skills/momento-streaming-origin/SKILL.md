---
name: momento-streaming-origin
description: Use Momento as a serverless data platform and streaming origin - account/cache/API-key setup via console.gomomento.com, the HTTP API for reading and writing cache data, publishing live HLS video with ffmpeg, TTL lifecycle, auth modes, browser playback, and latency tuning. Use when setting up Momento, storing/fetching data over its HTTP API, streaming video into or out of a cache, or debugging cache PUT/GET failures.
---

# Momento as a data platform and streaming origin

> **Guard - reference only. Loading this skill is not a request to act.**
> Do not create, build, deploy, or run anything based on this skill unless
> the user has explicitly asked for that work in this conversation. If they
> have not asked, answer from it and stop.

Momento is a serverless cache: you create a cache, get an API key, and
read/write items over HTTPS with per-item TTLs. Because every item is
addressable by URL, a cache can serve as a complete HTTP origin - including
a live-video origin, where the HLS playlist and segments are just cache
items. No servers to run; TTL is the garbage collector.

## Getting started (account, cache, key)

1. **Create an account** at https://console.gomomento.com (email, Google,
   or GitHub sign-in). The console is where everything starts.
2. **Create a cache.** Console: Caches -> Create Cache, pick a name and a
   cloud/region (e.g. AWS us-west-2). CLI: `momento cache create <name>`.
   The cache is the container for EVERYTHING: you cannot read or write
   data, and you cannot deploy a Momento Function, until a cache exists -
   functions deploy onto a specific cache
   (`/functions/manage/<cache>/<fn>`), and data lives at
   `/cache/<cache>?key=...`. Create the cache first; a missing cache turns
   every other call into an error.
3. **Generate an API key.** Console: API Keys section. Choose scope
   (super-user for development, fine-grained for production) and
   expiration. The key is a JWT; store it in a file (a common convention:
   one line in `~/.mo-creds`) and never commit it.
4. **Find your HTTP API endpoint.** It is per-cell and region-specific,
   e.g. us-west-2: `https://api.cache.cell-4-us-west-2-1.prod.a.momentohq.com`.
   The console shows endpoints alongside your key; if in doubt, probe with
   a test PUT (a wrong host fails DNS, the right one returns 204).

Smoke test the whole chain in one line:

```bash
KEY=$(cat ~/.mo-creds); EP=https://api.cache.<cell-host>
curl -X PUT -H "Authorization: $KEY" --data "hello" \
  "$EP/cache/<cache>?key=smoke-test&ttl_seconds=60"          # expect 204
curl -H "Authorization: $KEY" "$EP/cache/<cache>?key=smoke-test"  # "hello"
```

## HTTP API essentials

- Operations: `PUT/GET/DELETE https://<ep>/cache/<cache>?key=<key>`;
  PUT takes `&ttl_seconds=<n>` with the raw body as the value. GET returns
  the raw value; a miss is a 404 with a JSON body.
- Auth is EITHER the `Authorization: <api-key>` header OR a `&token=<key>`
  query param. Sending both returns 400.
- Request size limit: 1 MiB per item by default (429 "Resource Exhausted"
  above it; raisable per account). Design payloads around it.
- CORS: the API reflects any Origin, so browser pages hosted anywhere
  (S3, localhost) can fetch cache items directly. No proxy needed.
- Debug rule: when a client misbehaves against the API, reproduce with
  curl first. The API accepts even slow chunked uploads; client-side
  quirks (TLS connection handling, header formatting) are the usual
  culprit.

The cache works as a general JSON/data store this way too: store results,
rolling histories, and reference data (e.g. an embeddings library) as keys
next to whatever else lives there. One origin, one auth model, one
lifecycle.

## Publishing live HLS with ffmpeg

```
-f hls -hls_time 1 -hls_list_size 6 -hls_flags program_date_time \
-method PUT -http_persistent 1 \
-hls_segment_filename "https://<ep>/cache/<cache>?key=seg%05d.ts&ttl_seconds=60&token=<KEY>" \
"https://<ep>/cache/<cache>?key=stream.m3u8&ttl_seconds=30&token=<KEY>"
```

Non-negotiables:
- `-http_persistent 1` is REQUIRED. ffmpeg's connection-per-request HTTPS
  path dies mid-upload against the API (TLS broken pipe), and on some
  paths failures are silent. One persistent connection is reliable.
- Cap the bitrate (`-b:v 1500k -maxrate 1800k -bufsize 3600k` at 720p) so
  segments respect the 1 MiB item limit. Uncapped 720p produces ~1.2 MB
  segments that never land.
- Skip `delete_segments`: TTLs expire old segments (60s) and the playlist
  (30s) on their own. Stopping the stream empties the origin.
- If using header auth instead of token-in-URL, ffmpeg's `-headers` value
  needs a trailing CRLF.
- `program_date_time` stamps segments with wall clock - required for any
  downstream time alignment (metadata overlays, latency measurement).

Fallback when ffmpeg cannot PUT HTTPS directly (some static builds have
broken TLS/DNS): write HLS to a local directory and ship files with a curl
loop. In that mode add `+temp_file` to `-hls_flags` so segments appear
atomically via rename - without it the uploader races ffmpeg mid-write and
ships truncated segments that are never re-sent. Key hygiene for loops
like this: `curl -H @headerfile` keeps the API key off the process argv.

## Token-in-URL vs header (playback constraint)

Players resolve the playlist's RELATIVE segment URIs by replacing the query
string, so a header-authed playlist yields segment requests with no auth.
For browser/stock-player playback, put `&token=` in the segment filename
template so it is written into every playlist entry, then share the
playlist URL with the token. Trade-off: anyone who reads the playlist has
the token - scope and expire keys accordingly (the console can issue
fine-grained, short-lived keys; a Momento Function can even vend disposable
tokens on demand).

Playback then works everywhere:
- Browser: hls.js pointed at the playlist URL; Safari plays it natively.
- CLI: `ffplay '<playlist-url-with-token>'`.

## Latency budget

Momento adds milliseconds; HLS mechanics dominate. Measured ~3-5s glass to
glass with: 1s segments (`-g` = fps so every segment starts on a keyframe),
`hls_list_size 6`, and hls.js `liveSyncDurationCount: 2`,
`maxLiveSyncPlaybackRate: 1.05`, `startFragPrefetch: true`. The plain-HLS
floor is ~2x segment duration; below that requires LL-HLS (awkward on a
GET-only origin) or a different transport.

## Verification checklist

```bash
curl -s "https://<ep>/cache/<cache>?key=stream.m3u8&token=$KEY"   # 200, playlist
# media-sequence should advance ~1 per segment-duration:
grep MEDIA-SEQUENCE; sleep 5; grep MEDIA-SEQUENCE
# fetch a segment via a playlist-relative URI; first byte 0x47 (MPEG-TS):
curl -s "https://<ep>/cache/<seg-uri-from-playlist>" | head -c1 | od -An -tx1
```

## Useful CLI commands

```bash
momento configure                 # store credentials for the CLI
momento cache create <name>       # prerequisite for everything else
momento cache list
momento cache set/get --key k --value v   # quick data smoke tests
momento cache flush <name>        # clear a cache without deleting it
```
