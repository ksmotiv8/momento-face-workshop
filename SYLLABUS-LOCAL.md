# Local Track — Face Recognition with Native Inference + Valkey

The simpler sibling of `SYLLABUS.md`. Recognition runs as a native binary
on your machine instead of inside a Momento Function, and a local Valkey
instance indexes the embeddings for KNN search. You still build one
Momento Function (hello world) so the serverless model is not skipped,
and you can still stream through Momento as origin.

What this track trades away: the recognizer is no longer a shared service
(each client needs the models and compute), and you skip the wasm
constraints that make the full track hard - which is exactly why this
track is faster. What it adds: a real vector index instead of brute-force
matching.

**Prereqs:** Momento account + cache + API key (in `~/.mo-creds`) · Rust ·
Docker (for Valkey) · ffmpeg · the two models (README step 5) · skills
installed (README step 4).

Core track: Modules 0-4 and 6, about 3.5 hours. Module 5 is optional.

---

## Module 0 — Setup (20 min)

Momento side: cache + key + smoke test (same as the main track, Module 0).

Valkey side: run the bundle image, which includes the `search` module
(plain valkey has no FT.* commands):

```bash
docker run -d --name valkey --rm -p 6379:6379 valkey/valkey-bundle
docker exec valkey valkey-cli MODULE LIST | grep -A1 search   # expect: search
# if 6379 is taken, map another port: -p 16379:6379
```

> **Prompt:** "Verify my setup: Momento PUT/GET smoke test with the key in
> `~/.mo-creds` against cache `<cache>`, and confirm my local valkey at
> port 6379 has the search module loaded. Never echo the key."

---

## Module 1 — Hello World Momento Function (30 min)

Identical to the main track Module 1 (exercises 1a and 1b). Do not skip
it: it is the one place this track touches serverless compute, and the
deploy loop teaches the platform in 30 minutes.

---

## Module 2 — Face pipeline, native (60 min)

Identical to the main track Module 2: shared `facecore` crate, `libbuild`
CLI, threshold calibration with `faces/portraits` and `faces/probes`.
Native-only differences worth enjoying: model load is ~50 ms and each
embedding ~30 ms - no cold starts, no memory budget, full-size detection
planes.

The shipped `faces/faces-library.json` is your answer key: your `build`
output should match it closely (same model, same preprocessing).

---

## Module 3 — Valkey as the embeddings index (45 min)

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

**Exercise 3a — load the index**
> **Prompt:** "Write a `libbuild index` subcommand that loads
> `faces-library.json` into my local valkey as an HNSW COSINE index
> (FT.CREATE, one HSET per entry with the vector packed as little-endian
> f32 plus a name field). Then verify with an FT.SEARCH KNN query using
> one of the library vectors - it must return its own name at distance
> near zero."

**Exercise 3b — the recognizer**
> **Prompt:** "Write a `recognize` binary: given an image, detect faces
> with facecore, embed each, and match via FT.SEARCH KNN against valkey
> instead of brute force. Careful: valkey returns cosine DISTANCE
> (1 - similarity), so my similarity threshold of T accepts matches with
> dist <= 1 - T. Run the full probe set from `faces/probes` and produce
> the same table as the main track's test ladder: genuine pairs named,
> strangers unmatched."

*The distance-vs-similarity conversion is the lesson here. Port a
threshold across systems without checking the score convention and
recognition breaks silently in one direction or the other.*

---

## Module 4 — Live video, local recognition (40 min)

Same ffmpeg pattern as the main track (testsrc2 + face overlays on
co-prime periods, `split`, 2 fps tap to an atomically-rewritten JPEG),
but the sink calls your local `recognize` binary instead of POSTing to a
function.

> **Prompt:** "Wire the stream tap to my local recognizer: watch the
> tapped JPEG's mtime, run `recognize` on each new frame, and log capture
> time, per-frame latency, and names. Then PUT each result as JSON to my
> Momento cache under `faces.json` (ttl 60) so a remote dashboard could
> poll it. Report the latency distribution - it should be far below the
> function-based track's numbers."

*Expect tens of milliseconds per frame instead of hundreds. That gap is
the price and the payoff of the serverless version, made visible.*

---

## Module 5 — Momento as streaming origin (45 min, optional)

Unchanged from the main track Module 5. The origin story never depended
on where recognition runs: ffmpeg PUTs 1 s HLS segments to cache keys,
TTLs clean up, players pull straight from the cache.

---

## Module 6 — Enrollment (15 min)

No redeploy concept exists here - enrollment is just detect + embed +
HSET. The exercise is to notice how the properties moved: the main
track's "add a person with no redeploy" required the library-in-cache
design; locally it is nearly free, but only THIS machine learns the face.

> **Prompt:** "Add an `enroll <image> <name>` subcommand: detect the
> largest face, embed it, HSET it into the valkey index, and prove the
> very next `recognize` run names that person. Support multiple entries
> per person (face:<name>:<n> keys) - nearest entry wins."

---

## Module 7 — Write the blog post (45 min)

Same exercise as the main track Module 7, with one extra required
section: local vs serverless. You have now built both sides of the
tradeoff - report the numbers (per-frame latency, enrollment time,
operational surface) and say when you would choose each.

---

## Module 8 — Reflect: interrogate what you built (30 min)

Same exercise as the main track Module 8: stop asking the agent to do,
start asking it to explain, challenge, and quiz. All the prompts there
apply; add these two, which are specific to this track:

> We ran the same model in wasm (main track) and natively. Walk me
> through everything that changed and everything that did not - and what
> that says about where the real complexity in this system lives.

> Explain why valkey returns distance while our threshold was calibrated
> as similarity, and what would have happened if we had used 0.30 as a
> distance cutoff. What is the general lesson about porting numbers
> between systems?

---

## Reference deltas from the main track

| Topic | Main track | This track |
|---|---|---|
| Recognition runs | Momento Function (wasm) | native binary |
| Matching | brute-force dot product | Valkey FT.SEARCH KNN (HNSW) |
| Score convention | cosine similarity (higher = better) | cosine distance (lower = better); sim = 1 - dist |
| Library lives | cache key (JSON) | valkey index (+ JSON as answer key) |
| Enrollment | add-face function | `enroll` subcommand (HSET) |
| Per-face latency | ~360 ms | ~30 ms |
| Sharable service | yes | no (this machine only) |
| Models | same ONNX (w600k_mbf + SeetaFace), same tract runtime - keeps `faces-library.json` valid |
