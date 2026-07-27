# Momento Functions + Face Recognition — Workshop Syllabus

Serverless compute next to your data: WebAssembly functions running on Momento
hosts. You will build a hello-world function, then a face recognition service
whose ML models run in wasm and whose face library lives in the cache.

**Prereqs:** Momento account + cache + API key (in `~/.mo-creds`, one line) ·
`rustup target add wasm32-wasip2` · ffmpeg (modules 4–5) · your HTTP endpoint,
e.g. `https://api.cache.cell-4-us-west-2-1.prod.a.momentohq.com`

### Skills

This workshop assumes four skills are **registered and loaded** before you
start. A skill is a markdown reference the agent reads on demand — it does
not auto-execute, it just makes the agent informed. Load with `/skill <name>`:

| Skill | When it matters | What it gives you |
|---|---|---|
| `momento-functions` | Modules 1–3, 6 | Handler styles, deploy PUT shape, `OnceLock` warm-cache, env-var field name, 502/500 triage |
| `ffmpeg` | Modules 4–5 | `split` + `fps` frame tap, atomic JPEG rewrite, mtime-as-capture-time, HLS muxer settings |
| `momento-streaming-origin` | Module 5 | Cache-as-origin model, `-http_persistent 1`, token-in-URL playback, latency budget |
| `face-recognition` | Modules 2–3, 6 | Model choices, preprocessing contract, calibration procedure, test ladder, detection floor |

**Register the skills your platform needs ahead of the workshop.** If the
agent has to discover any of this from scratch, each module takes 2–3× longer
and produces more wrong turns. The prompts below assume the agent has read the
relevant skill; if a skill is missing, say so in the prompt or the agent will
guess — and guessing with credentials and deploys is expensive.

Each exercise has a **prompt** — paste it into your coding agent.

---

## Module 0 — Setup (15 min)

Create the cache first: nothing else works without it. Functions deploy *onto*
a cache (`/functions/manage/<cache>/<name>`), data lives at `/cache/<cache>?key=`.

Smoke test:
```bash
KEY=$(cat ~/.mo-creds); EP=https://api.cache.<cell-host>.prod.a.momentohq.com
curl -X PUT -H "Authorization: $KEY" --data "hello" \
  "$EP/cache/<cache>?key=smoke&ttl_seconds=60"     # expect 204
curl -H "Authorization: $KEY" "$EP/cache/<cache>?key=smoke"   # expect 200 with body "hello"
```

> **Prompt:** "Verify my Momento setup: my API key is in `~/.mo-creds` and my
> cache is `<name>` at endpoint `<endpoint>`. Do a PUT then GET of a test key
> and confirm both succeed. Never echo the key."

---

## Module 1 — Hello World function (30 min)

`cargo init --lib`, `crate-type = ["cdylib"]`, `.cargo/config.toml` with
`target = "wasm32-wasip2"`, deps `momento-functions-guest-web` +
`momento-functions-bytes`. Handler registered with `invoke!(handler)`; the
handler must take the request payload even if it ignores it:
`fn handler(_payload: Data) -> &'static str { ... }` (a zero-arg handler
fails with an opaque macro-expansion type error).
Deploy = base64 the wasm, `PUT /functions/manage/<cache>/hello`, expect 204.

**Exercise 1a — raw string**
> **Prompt:** "Create a Rust Momento Function that returns 'hello world' as a
> plain string. Build for wasm32-wasip2, deploy to cache `<name>` as `hello`,
> and invoke it to prove it works. Put the deploy+invoke steps in a reusable
> script that reads the key from `~/.mo-creds` inside the script."

**Exercise 1b — typed JSON**
> **Prompt:** "Change the function to accept `{"name":"..."}` and return
> `{"message":"Hello <name>"}` using `Json<T>` for request and response.
> Redeploy and test with my name."

**Two traps to know about:**
- Inlining base64 in the curl command → `Argument list too long`. Write the
  JSON body to a temp file, use `--data-binary @file`.
- Older runtimes rejected the trailing newline `echo '{"name":"x"}'` appends
  (400 *trailing characters*); current guest crates accept it, but
  `printf '%s'` or a file is still the safe habit.

*Expect: build ~0.7 s, deploy ~250 ms, first invoke ~100–250 ms, warm
invokes ~15–20 ms.*

---

## Module 2 — Face recognition, offline (60 min)

No training happens. A pretrained ArcFace model maps any face to a 512-D
vector; "learning" a person is storing one vector with a name.

**Models:** `seeta_fd_frontal_v1.0.bin` (1.2 MB detector, via `rustface`) and
`w600k_mbf.onnx` (13.6 MB embedder, via `tract`). Both pure-Rust inference —
that is what makes wasm feasible.

**Architecture rule:** put the pipeline in a **shared crate** used by both the
wasm function and a host CLI. Embeddings only compare if preprocessing is
byte-identical; one code path guarantees it.

**Exercise 2a — the shared core + host tool**
> **Prompt:** "Set up a Rust workspace with a shared `facecore` crate
> (detect → crop 112×112 chip → embed → L2-normalize → cosine match) and a
> host CLI `libbuild` with subcommands `build` (portraits dir → library JSON),
> `probe` (score images against the library) and `pairs` (all-pairs cosines).
> Use rustface for detection and tract-onnx for the ArcFace embedder. Verify
> it works natively before any wasm work."

**Exercise 2b — calibrate your own threshold**
> **Prompt:** "Download face photos with at least 2 different photos per
> person for 4+ people. Build a library from one photo each, then measure
> impostor cosines (`pairs`) and genuine cosines (probe the second photos).
> Report both ranges and recommend a threshold in the gap. Note: Wikimedia
> blocks cloud IPs — use raw.githubusercontent.com/ageitgey/face_recognition."

*Reference numbers: impostors ≤ 0.16, genuine 0.35–0.93, threshold 0.30.
Do not copy these — measure yours.*

**Key constraints:** `min_face_size` must be ≥ 20 or the detector panics (an
opaque 500 in wasm) · memory scales with decoded **pixels**, not JPEG bytes ·
`cargo update kstring@2.0.4 --precise 2.0.2` if rustc < 1.96.

---

## Module 3 — Deploy the recognizer (45 min)

The face library lives in the **cache**, not the wasm, so people can be added
without redeploying. Cache the parsed ONNX plan in a `OnceLock`: cold ~27 s,
warm ~620 ms.

**Exercise 3a — the function**
> **Prompt:** "Write a Momento Function that accepts a JPEG POST body and
> returns `{"faces":[{x,y,w,h,score,name,sim}]}`. Embed both models with
> `include_bytes!`, read the library from cache key `faces-library.json` on
> each invoke, and cache the parsed ONNX plan in a `OnceLock`. Bad input must
> return a clean 400, never a panic. Deploy it and PUT the library to cache."

**Exercise 3b — the test ladder** (each step isolates one layer)
> **Prompt:** "Test my deployed face function in this order and report a table:
> (1) garbage bytes — expect a clean 400, not a 500; (2) a library portrait —
> expect its own name near 0.98; (3) a different photo of the same person;
> (4) a multi-face photo; (5) strangers — expect faces with no name;
> (6) the same portrait 10× — confirm warm latency is stable."

**Exercise 3c — find the detection floor**
> **Prompt:** "Probe my function with a group photo containing many small
> faces. If it finds none, diagnose why by computing the face size after
> downscaling to the detection plane, then raise the plane via the
> `DETECT_PLANE` env var and re-test. Report faces found and latency at both
> settings."

*Expected finding: 0 faces at plane 512 (62 px faces → 20 px, at the floor);
25 faces at plane 1024, but ~35 s. This is the accuracy/latency/memory
tradeoff made concrete.*

---

## Module 4 — Live video recognition (45 min)

One ffmpeg process, two outputs from a `split`: an encoded stream plus a 2 fps
JPEG tap. `-update 1 -atomic_writing 1` rewrites **one** file atomically, so
readers never see a partial JPEG — and the file's mtime **is** the frame's
capture time.

**Exercise 4a — the stream + tap**
> **Prompt:** "Build a simulated camera stream with ffmpeg: `testsrc2`
> background with two known face images overlaid on co-prime periods (so they
> appear alone, together, and not at all). Split it: one branch encodes H.264,
> the other taps 2 fps to an atomically-rewritten JPEG. Verify the injected
> faces still recognize as stills *before* wiring up video."

**Exercise 4b — the sink**
> **Prompt:** "Write a sidecar that polls the tapped JPEG's mtime at 2× the
> tap rate, POSTs each new frame to my face function, and logs capture time,
> frame age, RTT and recognized names. Run ~24 frames and report latency
> broken down by number of faces in frame."

*Expected: ~195 ms for 0 faces, ~555 ms for 1, ~900 ms for 2 — about 360 ms
per face, dominated by the embedding pass. Note a serial sink self-limits to
one in-flight request, so it drops frames rather than saturating the server.*

---

## Module 5 — Momento as a streaming origin (45 min)

Every cache item is URL-addressable, so a cache **is** an HTTP origin. The HLS
playlist and segments are just cache items; TTL is the garbage collector.

**Exercise 5**
> **Prompt:** "Publish my ffmpeg stream as HLS directly to Momento as origin:
> 1 s segments PUT to cache keys with `-method PUT -http_persistent 1`, the
> playlist as another key. Cap the bitrate so segments stay under the 1 MiB
> item limit, set `-g` = fps so every segment starts on a keyframe, and put
> `&token=` in the segment filename template so players can resolve relative
> URIs. Verify by fetching the playlist and watching MEDIA-SEQUENCE advance,
> then fetch a segment and check the first byte is 0x47."

**Non-negotiables:** `-http_persistent 1` (without it, TLS dies mid-upload,
sometimes silently) · rate-cap for the 1 MiB limit · skip `delete_segments`,
let TTLs clean up · `program_date_time` for time alignment.

If ffmpeg reports "Failed to resolve hostname" while curl to the same host
works, your ffmpeg build's resolver is broken (common in static builds):
use the curl-loop fallback in the `momento-streaming-origin` skill.

*Latency budget: ~3–5 s glass-to-glass with 1 s segments and a player synced
2 segments from the live edge.*

---

## Module 6 — Enrollment without redeploy (30 min)

**Exercise 6**
> **Prompt:** "Write a second Momento Function `add-face` that takes a JPEG
> body plus `?name=<person>`, detects the largest face, embeds it, and appends
> it to `faces-library.json` in the cache. Then confirm my face-detect function
> recognizes the new person on the very next invoke, with no redeploy."

⚠️ **Known blocker — request before the workshop:** a 28 MB wasm (13.6 MB of
embedded ONNX) exceeds the default ~4 MB function memory budget — every invoke
returns 502 *"Function memory limit exceeded"*. Ask Momento support to raise
your function memory limit for the workshop account *before* Module 6.
Adding `memory_limit_mb` to the manage PUT is **untested** — if you verify it
works, update this syllabus.

---

## Module 7 — Write the blog post (45 min)

You built a face recognition service with no servers: models in wasm next to
a cache, a library that updates without redeploys, live video recognized in
flight. The last exercise is telling that story — and it doubles as review:
you cannot explain the pipeline without actually understanding it.

**Exercise 7**
> **Prompt:** "Write a blog post about what we built in this workshop, for an
> engineer who has never used Momento. Pull every number from THIS session's
> real measurements — deploy times, warm vs cold invoke latency, my measured
> impostor/genuine cosine ranges and threshold, per-face latency, segment
> sizes — not from the syllabus. Structure: the hook (what runs where, and
> what is NOT running), the architecture in one diagram, how recognition
> actually works (embeddings, not training), three things that went wrong and
> what each taught us, and what we would build next. Include real commands
> and responses where they carry the story. No em-dashes. Do not brag about
> platform internals — write what a reader can use."

**What makes it good:**
- Numbers from your session, not ours. If your cold invoke was 27 s, say
  27 s — the reader trusts a measured number more than a round one.
- The failures are the content. "502 memory limit exceeded" and the fix is
  more useful to a reader than a screenshot of success.
- One honest paragraph on limits (frontal-only detection, threshold measured
  on your data, memory budget) beats a caveat-free victory lap.

*Expect: the first draft overclaims and underspecifies. Ask the agent to
fact-check its own draft against the session transcript — every number, every
claim — before you call it done.*

---

## Module 8 — Reflect: interrogate what you built (30 min)

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

> I will explain how a frame gets from ffmpeg to a name on screen. Point
> out everything I get wrong or skip. [then type your explanation]

**Invert it:**
> If we rebuilt this from scratch tomorrow, what should we do differently?
> What did we learn the expensive way that we now get for free?

*The quiz usually stings. Wrong answers there are the most valuable
15 minutes of the workshop - each one is a thing you shipped without
understanding, found while it is still cheap to learn.*

---

## Reference

**Operational gotchas**
| Symptom | Cause |
|---|---|
| 502 memory limit exceeded | decoded pixels or model size vs budget |
| Bare 500, no message | wasm panic — configure topic logging early |
| Env vars empty | misspelled field; must be `environment_variables` (PUT still returns 204) |
| Recognition silently stops | library TTL expired → `library:0`, all faces unnamed |
| `Argument list too long` | base64 inline in curl; use `--data-binary @file` |
| ffmpeg exits at start | missing `-y`, prompting to overwrite |
| No `drawtext` filter | static build without libfreetype; use frame mtime instead |
| Detector panic | `min_face_size` < 20 |

**Costs to keep in view:** each face-function deploy ships ~36 MB of base64,
takes ~15 s, and cold invokes cost ~27 s. Deploy deliberately.

**Not covered:** Tier B (landmark alignment for tilted/profile faces — Tier A
is frontal only) · multiple photos per person · vector indexes beyond a few
hundred identities · server-side concurrency limits.
