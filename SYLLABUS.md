# Momento + Face Recognition - Workshop Syllabus

All in on faces and vectors: you will build a face recognition pipeline
that runs natively on your machine, index the face embeddings in a local
Valkey with real KNN vector search, recognize faces in live video, and
publish live results to a Momento cache - optionally streaming the video
itself through Momento as a serverless HLS origin. A hello-world Momento
Function ships as a take-home so the workshop hours stay on recognition.

**Prereqs:** Momento account + cache + API key (in `~/.mo-creds`, one line) ·
Rust · Docker (for Valkey) · ffmpeg (modules 3-4, full build not static) ·
the two models (README step 5) · your HTTP endpoint, e.g.
`https://api.cache.cell-4-us-west-2-1.prod.a.momentohq.com`

Core path: Modules 0-3, 5, and 6, about 3.75 hours. Module 4 is optional;
Module 7 is a recommended 30-minute wrap-up. The take-home adds ~30 min
whenever you like.

## Skills

This workshop assumes four skills are **registered and loaded** before you
start. A skill is a markdown reference the agent reads on demand - it does
not auto-execute, it just makes the agent informed.

| Skill | When it matters | What it gives you |
|---|---|---|
| `face-recognition` | Modules 1-3, 5 | Model choices, preprocessing contract, calibration procedure, detection floor |
| `ffmpeg` | Modules 3-4 | `split` + `fps` frame tap, atomic JPEG rewrite, mtime-as-capture-time, HLS muxer settings |
| `momento-streaming-origin` | Modules 0, 3, 4 | HTTP API, cache-as-origin model, `-http_persistent 1`, curl-loop fallback |
| `momento-functions` | Take-home | Handler styles, deploy PUT shape, `Data` payload signature, ARG_MAX trap |

**Register the skills ahead of the workshop.** If the agent has to discover
this from scratch, each module takes 2-3x longer and produces more wrong
turns. The prompts below assume the agent has read the relevant skill.

Each exercise has a **prompt** - paste it into your coding agent.

---

## Module 0 - Setup (20 min)

Momento side: create the cache first. It carries the live recognition
results in Module 3, serves as the streaming origin in Module 4, and the
take-home function deploys onto it. Data lives at `/cache/<cache>?key=`.

```bash
KEY=$(cat ~/.mo-creds); EP=<endpoint>   # e.g. https://api.cache.cell-4-us-west-2-1.prod.a.momentohq.com
curl -X PUT -H "Authorization: $KEY" --data "hello" \
  "$EP/cache/<cache>?key=smoke&ttl_seconds=60"     # expect 204
curl -H "Authorization: $KEY" "$EP/cache/<cache>?key=smoke"   # expect 200, body "hello"
```

Valkey side: run the bundle image, which includes the `search` module
(plain valkey has no FT.* commands):

```bash
ss -ltn | grep -E ':(6379|16379)\b'   # Linux: anything printed = port taken
# macOS: lsof -iTCP:6379 -sTCP:LISTEN
VALKEY_PORT=6379                      # set to any free host port
docker run -d --name valkey --rm -p ${VALKEY_PORT}:6379 valkey/valkey-bundle
docker exec valkey valkey-cli MODULE LIST | grep -A1 search   # expect: search
```

Every later valkey command in this workshop uses this same host port, so
note whatever you picked.

> **Prompt:** "Verify my setup: Momento PUT/GET smoke test with the key in
> `~/.mo-creds` against cache `<cache>` at endpoint `<endpoint>`, and
> confirm my local valkey at port `<port>` has the search module loaded.
> Never echo the key."

---

## Module 1 - Face pipeline, native (60 min)

No training happens. A pretrained ArcFace model maps any face to a 512-D
vector; "learning" a person is storing one vector with a name.

**Models:** `seeta_fd_frontal_v1.0.bin` (1.2 MB detector, via `rustface`) and
`w600k_mbf.onnx` (13.6 MB embedder, via `tract`). Both pure-Rust inference -
no system dependencies.

**Architecture rule:** put the pipeline in a **shared crate** consumed by
your CLI tools (and by anything you build on it later). Embeddings only
compare if preprocessing is byte-identical; one code path guarantees it.

**Exercise 1a - the shared core + host tool**
> **Prompt:** "Set up a Rust workspace with a shared `facecore` crate
> (detect -> crop 112x112 chip -> embed -> L2-normalize -> cosine match) and a
> host CLI `libbuild` with subcommands `build` (portraits dir -> library JSON),
> `probe` (score images against the library) and `pairs` (all-pairs cosines).
> Use rustface for detection and tract-onnx for the ArcFace embedder. Verify
> it works end to end on one portrait before building out the rest."

**Exercise 1b - calibrate your own threshold**
> **Prompt:** "Build a library from the portraits in `faces/portraits`, then
> measure impostor cosines (`pairs`) and genuine cosines (probe the
> second photos in `faces/probes`). Report both ranges and recommend a
> threshold in the gap."

*Reference numbers: impostors <= 0.16, genuine 0.35-0.93, threshold 0.30.
Do not copy these - measure yours.*

The shipped `faces/faces-library.json` is a sanity reference, not an
answer key. Internally consistent scores are what matter: check that your
same-person cosines sit well above your cross-person cosines. Your
embeddings will only match the shipped file exactly if your crop and
resize path matches the recipe byte for byte - see the caveat in
`faces/README.md`. Separation, not equality with the shipped file, is
the check.

**Key constraints:** `min_face_size` must be >= 20 or the detector panics ·
`cargo update kstring@2.0.4 --precise 2.0.2` if rustc < 1.96 · native perf
to enjoy: model load ~50 ms, ~30 ms per embedding, no cold starts, no
memory budget.

---

## Module 2 - Valkey as the embeddings index (45 min)

Replace brute-force matching with a vector index. Embeddings are stored
as HASH fields (packed little-endian float32) and searched with KNN:

```
FT.CREATE faces ON HASH PREFIX 1 face: SCHEMA v VECTOR HNSW 6
  TYPE FLOAT32 DIM 512 DISTANCE_METRIC COSINE
HSET face:0 v <packed-512-floats> name "Barack Obama"
FT.SEARCH faces "*=>[KNN 3 @v $q AS dist]" PARAMS 2 q <packed-query>
  RETURN 2 name dist DIALECT 2
```

(Verified against valkey-bundle: querying with a library vector returns
its own entry at dist ~0.0000 and the nearest impostor at ~0.86.)

Client note: if you talk to valkey from Rust, pin the `redis` crate
version in Cargo.toml (`redis = "=x.y.z"`) and write against that
version's API. The `Value` enum variants have been renamed more than
once across releases, so read your pinned version's enum in the registry
source rather than copying web snippets - they may not compile against
your version.

**Exercise 2a - load the index**
> **Prompt:** "Write a `libbuild index` subcommand that loads
> `faces-library.json` into my local valkey as an HNSW COSINE index
> (FT.CREATE, one HSET per entry with the vector packed as little-endian
> f32 plus a name field). Then verify with an FT.SEARCH KNN query using
> one of the library vectors - it must return its own name at distance
> near zero."

**Exercise 2b - the recognizer**
> **Prompt:** "Write a `recognize` binary: given an image, detect faces
> with facecore, embed each, and match via FT.SEARCH KNN against valkey
> instead of brute force. Careful: valkey returns cosine DISTANCE
> (1 - similarity), so my similarity threshold of T accepts matches with
> dist <= 1 - T. Run the full probe set from `faces/probes` and report a
> table: genuine pairs named, multi-face photos counted correctly,
> strangers unmatched."

*The distance-vs-similarity conversion is the lesson here. Port a
threshold across systems without checking the score convention and
recognition breaks silently in one direction or the other.*

---

## Module 3 - Live video, local recognition (40 min)

One ffmpeg process, two outputs from a `split`: an encoded stream plus a
2 fps JPEG tap. `-update 1 -atomic_writing 1` rewrites **one** file
atomically, so readers never see a partial JPEG - and the file's mtime
**is** the frame's capture time. A sidecar feeds each new frame to your
`recognize` binary, and the results land in your Momento cache where any
remote dashboard could poll them.

Have `recognize` locate its model files by absolute path (or relative to
the executable, not the working directory): the sidecar invokes it from
its own cwd, and a relative `../models/...` path makes every frame report
zero faces with the error hidden in swallowed stderr.

**Exercise 3a - the stream + tap**
> **Prompt:** "Build a simulated camera stream with ffmpeg: `testsrc2`
> background with two known face images overlaid on co-prime periods (so
> they appear alone, together, and not at all). Size the overlays so each
> face clears the 20 px detection floor at the tap resolution. Split it:
> one branch encodes H.264, the other taps 2 fps to an atomically-rewritten
> JPEG. Verify the injected faces still recognize as stills *before*
> wiring up video."

**Exercise 3b - the sidecar**
> **Prompt:** "Wire the stream tap to my local recognizer: watch the
> tapped JPEG's mtime, run `recognize` on each new frame, and log capture
> time, per-frame latency, and names. Confirm the log shows the
> alone/together/neither cases from the live stream. Then PUT each result
> as JSON to my Momento cache under `faces.json` (ttl 60) so a remote
> dashboard could poll it. Report the latency distribution."

*Expect tens of milliseconds per frame (measured: ~12/104/141 ms for
0/1/2 faces). For contrast, a serverless function-based recognizer
measured ~195/555/900 ms on the same test (see Reference) - that gap is
the price and the payoff of a shared service, made visible.*

---

## Module 4 - Momento as a streaming origin (45 min, optional)

Every cache item is URL-addressable, so a cache **is** an HTTP origin. The
HLS playlist and segments are just cache items; TTL is the garbage
collector.

**Exercise 4**
> **Prompt:** "Publish my ffmpeg stream as HLS directly to Momento as origin:
> 1 s segments PUT to cache keys with `-method PUT -http_persistent 1`, the
> playlist as another key. Cap the bitrate so segments stay under the 1 MiB
> item limit, set `-g` = fps so every segment starts on a keyframe, and put
> `&token=` in the segment filename template so players can resolve relative
> URIs. Verify by fetching the playlist and watching MEDIA-SEQUENCE advance,
> then fetch a segment and check the first byte is 0x47. For the `&token=`
> in the playlist, use a scoped short-lived key minted for this stream,
> not my main workshop key."

**Non-negotiables:** `-http_persistent 1` (without it, TLS dies mid-upload,
sometimes silently) · rate-cap for the 1 MiB limit · skip `delete_segments`,
let TTLs clean up · `program_date_time` for time alignment.

**Key hygiene:** the `&token=` in the playlist means anyone who reads the
playlist holds that key. Mint a scoped, short-lived key in the console for
the stream token; do not embed your super-user workshop key.

If ffmpeg reports "Failed to resolve hostname" while curl to the same host
works, your ffmpeg build's resolver is broken (common in static builds):
use the curl-loop fallback in the `momento-streaming-origin` skill.

*Latency budget: ~3-5 s glass-to-glass with 1 s segments and a player synced
2 segments from the live edge.*

---

## Module 5 - Enrollment (15 min)

Enrollment here is just detect + embed + HSET - nearly free. Notice how
the properties moved: a serverless recognizer needs a library-in-cache
design to add people without redeploying; locally there is nothing to
redeploy, but only THIS machine learns the face. Neither is better; they
answer different questions.

> **Prompt:** "Add a `libbuild enroll <image> <name>` subcommand: detect the
> largest face, embed it, HSET it into the valkey index, and prove the
> very next `recognize` run names that person. Support multiple entries
> per person (face:<name>:<n> keys) - nearest entry wins."

---

## Module 6 - Write the blog post (45 min)

You built a system worth writing about: a native ML pipeline, a real
vector index, live video recognition, results in a serverless cache. The
last build exercise is telling that story - and it doubles as review:
you cannot explain the pipeline without actually understanding it.

**Exercise 6**
> **Prompt:** "Write a blog post about what we built in this workshop, for an
> engineer who has never used Momento or a vector index. Pull every number
> from THIS session's real measurements - my measured impostor/genuine
> cosine ranges and threshold, KNN distances, per-frame latency - not from
> the syllabus. Structure: the hook (what runs where, and what is NOT
> running), the architecture in one diagram, how recognition actually
> works (embeddings, not training), a local-vs-serverless section (use the
> syllabus Reference numbers for the serverless side, or your own if you
> built the function-based recognizer from the face-recognition skill), three things that went wrong and what each taught
> us, and what we would build next. Include real commands and responses
> where they carry the story. No em-dashes. Do not brag about platform
> internals - write what a reader can use."

**What makes it good:**
- Numbers from your session, not ours. The reader trusts a measured number
  more than a round one.
- The failures are the content. A resolver error and its fix is more
  useful to a reader than a screenshot of success.
- One honest paragraph on limits (frontal-only detection, threshold
  measured on your data, single-machine recognizer) beats a caveat-free
  victory lap.

*Expect: the first draft overclaims and underspecifies. Ask the agent to
fact-check its own draft against the session transcript - every number,
every claim - before you call it done.*

---

## Module 7 - Reflect: interrogate what you built (30 min)

You built it with an agent, which means you can build something without
fully understanding it. This module closes that gap. The technique:
instead of asking the agent to DO things, ask it to explain, challenge,
and quiz - the agent has full context of your session, which makes it the
best tutor you will ever have for this exact system.

Run these one at a time. Do not skim the answers; argue with them.

**Understand the design:**
> Walk me through every design decision we made that I did not explicitly
> ask for. For each: what was the alternative, and why did you pick this?

> Why do embeddings work for recognition at all? Explain what the 512
> numbers mean, why cosine similarity is the right comparison, and what
> the L2 normalization is for - to someone who knows no ML.

**Probe the limits:**
> What breaks first if 100 cameras hit this system at once? Walk the
> request path and find the bottleneck.

> What are three ways this system misidentifies someone, and what would
> each cost in a real deployment?

**Test yourself:**
> Quiz me: 10 questions about what we built, one at a time, hardest
> first. Wait for my answer before showing yours. Keep score honestly.

> I will explain how a frame gets from ffmpeg to a name in the log. Point
> out everything I get wrong or skip. [then type your explanation]

**About this build:**
> Our ML runs natively on this machine. Walk me through what would change -
> and what would not - if the recognizer moved inside a Momento Function
> as a shared service. What does that say about where the real complexity
> in this system lives?

> Explain why valkey returns distance while our threshold was calibrated
> as similarity, and what would have happened if we had used 0.30 as a
> distance cutoff. What is the general lesson about porting numbers
> between systems?

**Invert it:**
> If we rebuilt this from scratch tomorrow, what should we do differently?
> What did we learn the expensive way that we now get for free?

*The quiz usually stings. Wrong answers there are the most valuable
15 minutes of the workshop - each one is a thing you shipped without
understanding, found while it is still cheap to learn.*

---

## Take-home - Hello World Momento Function (30 min)

The serverless-compute side of Momento, kept out of the workshop hours so
they stay on recognition. Do this whenever you like; it needs only your
existing cache and key, plus the wasm target:
`rustup target add wasm32-wasip2`. Load the `momento-functions` skill
first.

Scaffold: `cargo init --lib`, `crate-type = ["cdylib"]`, `.cargo/config.toml`
with `target = "wasm32-wasip2"`, deps `momento-functions-guest-web` +
`momento-functions-bytes`. Handler registered with `invoke!(handler)`; the
handler must take the request payload even if it ignores it:
`fn handler(_payload: Data) -> &'static str { ... }` (a zero-arg handler
fails with an opaque macro-expansion type error).
Deploy = base64 the wasm, `PUT /functions/manage/<cache>/hello`, expect 204.

**Exercise A - raw string**
> **Prompt:** "Create a Rust Momento Function that returns 'hello world' as a
> plain string. Build for wasm32-wasip2, deploy to cache `<cache>` as `hello`,
> and invoke it to prove it works. Put the deploy+invoke steps in a reusable
> script that reads the key from `~/.mo-creds` inside the script."

**Exercise B - typed JSON**
> **Prompt:** "Change the function to accept `{"name":"..."}` and return
> `{"message":"Hello <name>"}` using `Json<T>` for request and response.
> Redeploy and test with my name."

**Two traps to know about:**
- Inlining base64 in the curl command -> `Argument list too long`. Write the
  JSON body to a temp file, use `--data-binary @file`.
- Older runtimes rejected the trailing newline `echo '{"name":"x"}'` appends
  (400 *trailing characters*); current guest crates accept it, but
  `printf '%s'` or a file is still the safe habit.

*Expect: build ~0.7 s, deploy ~250 ms, first invoke ~100-250 ms, warm
invokes ~15-20 ms.*

**Going further:** the `face-recognition` skill documents running the whole
recognizer inside a Momento Function - detector and embedder compiled to
wasm, library in the cache, a shared service any client can call.

---

## Reference

**Operational gotchas**
| Symptom | Cause |
|---|---|
| No FT.* commands in valkey | plain valkey image; use `valkey/valkey-bundle` (search module) |
| Valkey container will not start | host port taken; map another (`-p <free>:6379`) |
| Rust valkey client will not compile | `redis` crate `Value` enum renamed across versions; pin and read your version's source |
| Everything matches / nothing matches after porting | distance vs similarity convention; sim = 1 - dist |
| Sidecar reports 0 faces every frame | relative model paths resolved from the wrong cwd |
| ffmpeg "Failed to resolve hostname" (curl works) | static build resolver; use the curl-loop fallback |
| ffmpeg exits at start | missing `-y`, prompting to overwrite |
| Detector panic | `min_face_size` < 20 |
| `Argument list too long` on function deploy (take-home) | base64 inline in curl; use `--data-binary @file` |
| Zero-arg handler will not compile (take-home) | `invoke!` handlers must take the payload: `fn h(_p: Data)` |

**Serverless comparison (measured on a function-based build of the same
recognizer):** per-face latency ~360 ms vs ~30 ms native; cold invoke ~27 s
(13.6 MB model parse in wasm) vs ~50 ms native model load; in exchange the
function version is a shared service any client can call, with the library
updatable in the cache. The `face-recognition` skill documents that full
recipe if you want to build it.

**Not covered:** landmark alignment for tilted/profile faces (detection is
frontal-only) · HNSW tuning and vector-index scaling · running the
recognizer as a shared service · server-side concurrency limits.
