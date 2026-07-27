---
name: face-recognition
description: Reference for face detection and recognition inside Momento Functions - detector tiers, ArcFace-family embeddings via tract/ONNX in wasm, reference libraries in the cache, threshold calibration, test data sourcing, deploy/test procedure. Use ONLY when the user explicitly asks to build, add, or debug face recognition or detection. Do not proactively suggest or start building this pipeline.
---

# Face Recognition in a Momento Function

> **Guard - reference only. Loading this skill is not a request to act.**
> Do not create, build, deploy, or run anything based on this skill unless
> the user has explicitly asked for that work in this conversation. If they
> have not asked, answer from it and stop.

Build a serverless face recognition service: POST a JPEG to a Momento
Function, get back face locations and names. The detector, the embedding
model, and the matching all run in WebAssembly next to your cache. The
reference library of known faces lives in the cache itself, so you can add
people without redeploying.

```
JPEG  ──POST──►  Momento Function "face-detect"
                  1. detect faces        (detector model in wasm)
                  2. align + embed each  (ArcFace-family ONNX via tract)
                  3. match vs library    (cosine similarity)
                  ▼
                 {"faces":[{"x":..,"y":..,"w":..,"h":..,"name":"Barack Obama","sim":0.94}]}
                  + results written to cache keys for other consumers
```

The architecture is general: any detector + any embedding model with a
pure-Rust inference path fits this shape, and the pattern extends beyond
faces to any "small model + reference set" recognition task.

## Choose your tier

Two configurations of the same pipeline. Be deliberate about which one
your application needs:

**Tier A - frontal baseline.** Detector: SeetaFace via the `rustface`
crate (pure Rust, frontal faces only). No landmark alignment; crop the
detection box with a margin. This is the fully verified recipe in this
doc: simple, small, and reliable when subjects face the camera - webcams,
kiosk check-in, ID-style photos, controlled streams.

**Tier B - general purpose.** Detector: a modern ONNX face detector that
also outputs 5 facial landmarks (SCRFD or YuNet; both are small enough
for wasm and run under tract). Add landmark alignment before embedding.
Required for in-the-wild imagery: tilted heads, profiles, partial
occlusion, surveillance-style footage. The embedding and matching stages
are identical; only detection and preprocessing change.

The frontal baseline is not a toy - it is the right tool when you control
the camera. But do not ship it against uncontrolled imagery and conclude
"face recognition does not work" when it misses a turned head.

## Prerequisites

- A Momento account (console.gomomento.com), a cache (us-west-2
  recommended), and an API key. The cache must exist before you can
  deploy a function.
- Rust with the wasm target: `rustup target add wasm32-wasip2`.
- Models (download once into your crate directory):
  - Embedder, both tiers: w600k_mbf (~13.6 MB), MobileFaceNet trained
    with ArcFace on WebFace600K, 112x112 in, 512-D out:
    `https://huggingface.co/immich-app/buffalo_s/resolve/main/recognition/model.onnx`
    Any ArcFace-family ONNX embedder works the same way; this one is a
    good size/accuracy point for wasm.
  - Tier A detector: SeetaFace frontal (~1.2 MB):
    `https://github.com/atomashpolskiy/rustface/raw/master/model/seeta_fd_frontal_v1.0.bin`
  - Tier B detector: SCRFD (e.g. `det_500m.onnx` from the same buffalo_s
    pack) or YuNet from the OpenCV zoo; verify tract parses your pick
    before committing.

Why these choices: everything has a pure-Rust inference path (rustface
for SeetaFace, tract for ONNX), which is what makes wasm feasible. No C
dependencies, no runtime downloads.

## Step 1: Scaffold the function

```bash
cargo init --lib face-detect-fn && cd face-detect-fn
```

`Cargo.toml`:

```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
momento-functions-guest-web = "0"
momento-functions-bytes     = "0"
momento-functions-cache     = "0"
rustface   = { version = "0.1", default-features = false }  # tier A; no rayon in wasm
tract-onnx = "0.21"
image      = { version = "0.25", default-features = false, features = ["jpeg", "png"] }
serde      = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
opt-level = 3
lto = true
strip = true
```

`.cargo/config.toml`:

```toml
[build]
target = "wasm32-wasip2"
```

Embed the models in the binary:

```rust
static DETECT_MODEL: &[u8] = include_bytes!("../seeta_fd_frontal_v1.0.bin");
static EMBED_ONNX:   &[u8] = include_bytes!("../w600k_mbf.onnx");
```

A wasm with both models is ~26 MB; the deploy endpoint accepts it.

## Step 2: Detection

Tier A (verified):

```rust
let model = rustface::read_model(std::io::Cursor::new(DETECT_MODEL))?;
let mut detector = rustface::create_detector_with_model(model);
detector.set_min_face_size(20);          // MUST be >= 20 (the model's window)
detector.set_score_thresh(2.0);
detector.set_pyramid_scale_factor(0.8);  // use 0.6 under tight memory
detector.set_slide_window_step(4, 4);

let gray = img.to_luma8();
let (w, h) = gray.dimensions();
let mut data = rustface::ImageData::new(&gray, w, h);
let faces = detector.detect(&mut data);  // Vec<FaceInfo>: bbox() + score()
```

Tier B: run your detector ONNX through tract exactly like the embedder in
Step 3 (parse once behind a `OnceLock`, feed the expected input size,
decode its output boxes + landmarks). SCRFD and YuNet both output
per-face 5-point landmarks, which Step 3a consumes.

Rules that apply to both tiers:

1. In wasm a panic is a trap: the caller sees an opaque 500. Guard your
   invariants (tier A example: `min_face_size` below 20 makes the
   pyramid upscale and panic).
2. Memory scales with decoded PIXELS and the detection plane, not JPEG
   bytes. Functions have a per-account memory budget (4 MB default; CV
   workloads want it raised). Downscale large inputs to a fixed
   detection plane and scale the boxes back to original coordinates.
   Size the plane to your subjects: frames from a controlled stream can
   detect at 256-320 px; photos with small background faces need a larger
   plane and a correspondingly larger memory budget.

## Step 3: Embedding

Input format contract first: decode with `image::load_from_memory` (it
sniffs the format from magic bytes) and enable both `jpeg` and `png`
features, so the function and every host tool accept identical formats. A
function that hard-codes `ImageFormat::Jpeg` silently rejects PNG test
probes with a decode error while the host CLI accepts them - the two sides
then disagree about the same test set. If you deliberately restrict
formats, restrict everywhere and say so in the error message.

The preprocessing contract. Every detail must match on any system that
produces embeddings you intend to compare (see Step 4):

1. Obtain a 112x112 RGB face chip (Step 3a).
2. Normalize: `(pixel - 127.5) / 127.5`, laid out NCHW `1x3x112x112`.
3. Run the model; take the 512-float output.
4. L2-normalize the vector. Cosine similarity between unit vectors is a
   plain dot product, which keeps matching trivial.

### Step 3a: Getting the face chip

Tier A (crop, verified): expand the detection box by 25% to a square,
clamp to the frame, crop, resize to 112x112. Adequate for upright,
frontal faces.

Tier B (align, canonical): warp the face so its landmarks land on the
ArcFace reference template, then the chip is 112x112 by construction.
Compute a similarity transform (rotation + uniform scale + translation,
solvable in closed form / least squares) from the detected 5 points to
the template, and apply it with a warp. The standard 112x112 template:

```
left eye   (38.2946, 51.6963)
right eye  (73.5318, 51.5014)
nose tip   (56.0252, 71.7366)
left mouth (41.5493, 92.3655)
right mouth(70.7299, 92.2041)
```

Alignment is what makes embeddings robust to head tilt and pose, and it
is required if you want scores comparable to published ArcFace numbers
(which also assume BGR channel order; pick one order and use it
everywhere).

### The embedding code (both tiers)

```rust
use std::sync::OnceLock;
use tract_onnx::prelude::*;

fn embedder() -> &'static TypedRunnableModel<TypedModel> {
    static M: OnceLock<TypedRunnableModel<TypedModel>> = OnceLock::new();
    M.get_or_init(|| {
        tract_onnx::onnx()
            .model_for_read(&mut std::io::Cursor::new(EMBED_ONNX)).expect("parse")
            .with_input_fact(0, f32::fact([1, 3, 112, 112]).into()).expect("fact")
            .into_optimized().expect("optimize")
            .into_runnable().expect("plan")
    })
}

fn embed(chip: &image::RgbImage) -> TractResult<Vec<f32>> {
    let input: Tensor = tract_ndarray::Array4::from_shape_fn(
        (1, 3, 112, 112),
        |(_, c, y, x)| (chip.get_pixel(x as u32, y as u32)[c] as f32 - 127.5) / 127.5,
    ).into();
    let out = embedder().run(tvec!(input.into()))?;
    let raw: Vec<f32> = out[0].as_slice::<f32>()?.to_vec();
    let norm = raw.iter().map(|v| v * v).sum::<f32>().sqrt();
    Ok(raw.into_iter().map(|v| v / norm).collect())
}
```

The `OnceLock` is not optional polish. Parsing 13.6 MB of ONNX in wasm
takes ~6.5 seconds; Momento reuses warm instances, so caching the parsed
plan turns every invoke after the first into ~0.5 seconds total.

## Getting test data (portraits and probes)

You need two kinds of images: portraits of known people to build the
library, and probe images to test recognition. A ready-made set (5
identities, genuine-pair probes, multi-face photos, strangers) ships in
the workshop repo:
https://github.com/ksmotiv8/momento-face-workshop/tree/main/faces

**Wikipedia lead portraits** (good demo library): the REST API gives one
canonical portrait per public figure, with stable URLs. Send a real
User-Agent and keep the thumbnail URL exactly as returned (do not rewrite
the size in the path):

```bash
for NAME in Barack_Obama Joe_Biden Kit_Harington Rose_Leslie Alex_Lacamoire; do
  URL=$(curl -sL "https://en.wikipedia.org/api/rest_v1/page/summary/$NAME" \
        -H "User-Agent: my-face-demo/1.0 (you@example.com)" \
        | python3 -c "import json,sys; print(json.load(sys.stdin).get('thumbnail',{}).get('source',''))")
  curl -sL -H "User-Agent: my-face-demo/1.0 (you@example.com)" -o "${NAME//./}.jpg" "$URL"
done
```

Caveat: Wikimedia blocks requests from many cloud-provider IP ranges at
the network layer, so this loop 403s instantly from EC2 and similar hosts.
Run it from a laptop, or use sources that work from cloud machines
(GitHub raw files, HuggingFace mirrors).

The classic LFW image host at vis-www.cs.umass.edu is offline; do not
plan around per-image LFW URLs. Bulk mirrors exist on Kaggle and
HuggingFace if you want volume - and for tier B validation you WANT
volume: multiple photos per person across pose and lighting.

**Known-good probes**:

- Group photo (many small faces; stress-tests the detection floor):
  `https://github.com/atomashpolskiy/rustface/raw/master/assets/test/scientists.jpg`
- Single large portrait (easy positive):
  `https://raw.githubusercontent.com/opencv/opencv/master/samples/data/lena.jpg`
- Your own webcam:
  `ffmpeg -f avfoundation -framerate 30 -i "0" -frames:v 1 me.jpg` (macOS),
  `ffmpeg -f v4l2 -i /dev/video0 -frames:v 1 me.jpg` (Linux).

**Derive variants** to probe limits:

```bash
ffmpeg -i src.jpg -vf scale=256:-2 -q:v 4 probe-256.jpg
ffmpeg -i src.jpg -vf "crop=W:H:X:Y,scale=320:-1" face-crop.jpg
```

Data-quality rules that matter more than volume:

1. Library portraits should have the face filling a good fraction of the
   frame; the pipeline runs on a cropped face chip.
2. Respect the detection floor when probes get downscaled: SeetaFace
   needs ~20 px of face. A face filling a third of a 256 px frame is
   ~85 px (easy); a background face can be under 20 px (invisible to the
   detector, obvious to you). Test with tight crops before concluding the
   model is broken.
3. Your test set must include what production will see. Frontal-only
   probes validate tier A; they say nothing about tilted or profile
   faces.

## Step 4: Build the library (on your workstation)

Structure the project as a Cargo workspace with a shared core crate
(detect -> chip -> embed -> match) consumed by BOTH the wasm function and
a host CLI. This makes the critical rule structural instead of a matter of
discipline: embeddings only compare if preprocessing is byte-identical
(detector settings, chip extraction, normalization, channel order), and
one shared code path guarantees it. The host CLI turns a folder of
portraits into the library file and doubles as the calibration harness.

For each portrait: detect the biggest face, extract the chip, embed,
L2-normalize, and record `{ "name": "...", "e": [512 floats] }`. Store
the array in the cache so the function reads it at invoke time and you
can update it without redeploying:

```bash
curl -X PUT -H "Authorization: $KEY" --data-binary @faces-library.json \
  "https://<endpoint>/cache/<cache>?key=faces-library.json&ttl_seconds=86400"
```

The library is a cache item, so it EXPIRES. When it does, recognition
degrades silently: every face comes back unnamed and nothing errors. Two
mitigations: echo the library size in every response (a sudden
`"library": 0` is the tell), and refresh the TTL - re-PUT on a schedule,
or let an enrollment function rewrite the key (which resets it).

For robustness, store 2-3 embeddings per person (different photos) and
match against all of them, or average them into a centroid. A 512-float
entry is ~6 KB of JSON; brute-force matching stays fast into the
hundreds of entries. Beyond that, move to a vector index (the
momentohq/functions examples cover Turbopuffer and Valkey KNN).

## Step 5: Match

```rust
fn best_match<'a>(library: &'a [LibEntry], e: &[f32]) -> Option<(&'a str, f32)> {
    library.iter()
        .map(|l| (l.name.as_str(), l.e.iter().zip(e).map(|(a, b)| a * b).sum::<f32>()))
        .max_by(|a, b| a.1.total_cmp(&b.1))
        .filter(|(_, cos)| *cos >= MATCH_THRESHOLD)
}
```

**Do not copy a threshold; measure one.** The procedure:

1. Compute all-pairs cosines across your library: different people should
   cluster near 0 (with this model family, roughly <= 0.2).
2. Compute genuine pairs: multiple photos of the same people, across the
   pose/lighting range you expect in production.
3. Put the threshold in the gap, biased toward your failure preference
   (higher = fewer false accepts, more misses).

Calibration points from a frontal, well-lit test set (six Wikipedia
portraits, near-frontal probes): impostor pairs measured <= 0.12, genuine
pairs 0.78-0.97, and 0.35 sat comfortably between. Treat those numbers as
what an EASY dataset looks like, not a promise: genuine pairs across hard
pose/lighting changes can fall to 0.3-0.5, which is exactly why tier B
alignment and a measured threshold matter for general use. Also test the
open-set case: probe with strangers (plural) and verify they fall below
threshold rather than matching the nearest library entry.

In the handler: load the library from the cache
(`cache::get("faces-library.json")`), then for each detected face:
extract chip, embed, match, attach `name`/`sim`. Include the library size
and active threshold in every response - they make silent failure modes
(expired library, misconfigured env var) visible in one glance. Write the
result JSON to a cache key (with a TTL) so other consumers can poll it.

## Step 6: Deploy and test

```bash
cargo build --release
# Write the deploy body to a FILE: this wasm is ~26 MB, and inlining the
# base64 into curl argv hits ARG_MAX ("Argument list too long").
base64 -w0 target/wasm32-wasip2/release/face_detect.wasm > /tmp/fn.b64
printf '{"inline_wasm":"' > /tmp/fn.json; cat /tmp/fn.b64 >> /tmp/fn.json; printf '"}' >> /tmp/fn.json
curl -X PUT "https://<endpoint>/functions/manage/<cache>/face-detect" \
  -H "authorization: $KEY" -H "Content-Type: application/json" \
  --data-binary @/tmp/fn.json        # expect 204
```

Test in this order; each step isolates a layer:

```bash
# 1. Garbage input: expect a clean decode error, NOT a 500.
#    Proves the function instantiates and error handling works.
curl -X POST -H "authorization: $KEY" --data "notanimage" \
  "https://<endpoint>/functions/<cache>/face-detect"

# 2. A library portrait: expect its own name at high similarity (>0.9),
#    since it matches the exact image the library was built from.
# 3. A different photo of the same person: genuine-pair score; record it.
# 4. Strangers (several): expect faces with no name attached.
# 5. Repeat #2 ten times: all 200s, fast after the first (warm path).
# 6. Tier B only: tilted/profile probes; expect detection AND correct
#    matches where tier A would miss entirely.
```

Triage:
- 502 "Function memory limit exceeded": input too large or budget too
  small. Shrink the input, downscale before detecting, use coarser
  pyramid steps, or request a higher limit.
- Bare 500: suspect a panic. Configure topic logging
  (`momento-functions-host-log`) and tail it, or bisect with payloads.

## Performance summary (measured, tier A configuration)

| Metric | Value |
|---|---|
| wasm size (detector + embedder embedded) | ~26 MB |
| Cold invoke (ONNX parse) | ~6.5 s |
| Warm invoke (detect + embed + match) | ~0.5 s |
| Library entry | ~6 KB JSON per person |
| Latency scaling | roughly linear per face (~360 ms/face at a 512 plane; the embedding pass dominates) |

The rustface `Detector` is stateful (`detect` takes `&mut self`) and
cannot be cached in a `OnceLock` the way the embedder plan is. Cache the
parsed `rustface::Model` (it is `Clone`) in a `OnceLock` and construct a
detector from it per call - model parsing is the expensive part;
detector construction is cheap.

## Going further

- **Live video**: feed the function frames from a stream (ffmpeg frame
  tap: `fps=2,scale=256:-2` into an atomically rewritten JPEG, POST on
  mtime change). Pass the frame's capture time in a header so results can
  be time-aligned with playback.
- **Scale**: past a few hundred identities, move matching to a vector
  index. See the turbopuffer-* and valkey-cluster-vector-* examples in
  https://github.com/momentohq/functions/tree/main/examples.
- **Beyond faces**: the same skeleton (detector -> chip -> embedding ->
  cosine vs cached library) works for logos, products, or any
  open-vocabulary recognition where a small ONNX embedder exists.
