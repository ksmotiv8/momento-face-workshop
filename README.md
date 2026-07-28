# Momento Face Recognition Workshop

Build a face recognition system with a coding agent: a native ML pipeline
detects and embeds faces, a local Valkey indexes the embeddings with real
KNN vector search, live video gets recognized in flight, and results land
in a Momento cache - which can optionally serve the video itself as a
serverless HLS origin. A hello-world Momento Function ships as a take-home
for the serverless-compute side.

```
camera / test stream ──ffmpeg──► Momento cache (results + optional HLS origin)
                        │
                        └─frames──► recognize (native binary)
                                    detect -> embed -> FT.SEARCH KNN -> names
                                                          │
                                                    local Valkey index
```

You drive the build by prompting a coding agent. This repo gives the agent
the knowledge it needs (the `skills/`), gives you the exercises
(`SYLLABUS.md`, `PROMPTS.md`), and gives the pipeline test data (`faces/`).

## Repo layout

The workshop: face recognition running natively with a local Valkey
indexing the embeddings (real KNN vector search), live-video recognition,
and optionally Momento as the HLS streaming origin. A hello-world Momento
Function is a take-home follow-up. Validated end to end by autonomous
agent runs. (Want recognition running *inside* a Momento Function? The
`face-recognition` skill documents that full recipe as an extension.)

```
SYLLABUS.md          the workshop: 8 modules + a take-home, with prompts and timings
PROMPTS.md           all prompts in one copy-paste sheet
skills/              agent reference skills (install into your agent, step 4)
  momento-functions/         write, deploy, debug wasm functions
  momento-streaming-origin/  the cache as data store and HLS origin
  ffmpeg/                    capture, streaming, overlays, frame taps
  face-recognition/          the full detect/embed/match recipe
faces/               training portraits and test probes (see faces/README.md)
```

## Step-by-step setup

### 1. Create a Momento account

1. Sign up at https://console.gomomento.com (email, Google, or GitHub).
2. Create a cache: Caches -> Create Cache. Pick AWS `us-west-2`
   (recommended: it is where new capabilities land first). Name it
   anything; the syllabus calls it `<cache>`.
3. Create an API key: API Keys -> generate. Super-user scope is fine for
   the workshop. Copy the key.
4. Save the key where the exercises expect it, and never commit it:

```bash
printf '%s' '<paste-your-key>' > ~/.mo-creds
chmod 600 ~/.mo-creds
```

5. Note your HTTP endpoint. For us-west-2 it is
   `https://api.cache.cell-4-us-west-2-1.prod.a.momentohq.com`
   (other regions: https://docs.momentohq.com/platform/regions).

The cache must exist before anything else works: all data lives at
`/cache/<cache>?key=...`, and the take-home function deploys onto it.

### 2. Install the tools

```bash
# Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32-wasip2   # only needed for the take-home function

# Docker (runs the local Valkey; the workshop uses the valkey/valkey-bundle
# image because it includes the search module - plain valkey has no FT.*)
# macOS: https://docs.docker.com/desktop/   Linux: your distro's docker packages

# ffmpeg (modules 3-4) - use a full build, not a static one
brew install ffmpeg            # macOS
sudo apt install ffmpeg        # Debian/Ubuntu
sudo dnf install ffmpeg        # RHEL/Amazon Linux (enable EPEL/RPM Fusion first)
```

Why not a static ffmpeg build: common static builds (e.g. the johnvansickle
tarballs) ship a broken DNS/TLS resolver and fail Module 4's direct HTTPS PUT
with "Failed to resolve hostname ... System error" even though curl works. If
a static build is all you can get, use the curl-loop fallback documented in
the `momento-streaming-origin` skill.

```bash
# Momento CLI (optional but handy: cache create/list/set/get)
brew install momentohq/tap/momento-cli          # macOS
# or grab a release: https://github.com/momentohq/momento-cli/releases
```

### 3. Install a coding agent

Either works; the skills are compatible with both.

- **mo** (Momento's native coding agent):

  ```bash
  brew install momentohq/tap/mo
  mo login        # browser OAuth
  ```

  mo reads project skills from `.mo/skills/` and also honors Claude Code
  skills. It self-updates on launch.
- **Claude Code**: `npm install -g @anthropic-ai/claude-code`, then
  `claude` in this directory. Claude Code reads project skills from
  `.claude/skills/`.

### 4. Install the skills

From the repo root, copy the skills to where your agent looks:

```bash
# for mo
mkdir -p .mo/skills && cp -r skills/* .mo/skills/

# for Claude Code
mkdir -p .claude/skills && cp -r skills/* .claude/skills/
```

Skills are reference material the agent reads on demand. They carry a
guard: loading one never starts work by itself. But they matter a lot:
without them, each module takes 2-3x longer while the agent rediscovers
platform behavior (deploy payload shapes, the Valkey distance-vs-similarity
convention, client version pinning, HLS quirks) the hard way.

### 5. Download the models

The two ML models are not in this repo (size and licensing). Download them
once into a stable `models/` directory now; the crate the agent creates in
Module 1 will reference them from there (e.g. `../models/`), so they
survive any later restructuring:

```bash
mkdir -p models

# SeetaFace frontal detector (~1.2 MB)
curl -L -o models/seeta_fd_frontal_v1.0.bin \
  https://github.com/atomashpolskiy/rustface/raw/master/model/seeta_fd_frontal_v1.0.bin

# w600k_mbf ArcFace embedder (~13.6 MB, InsightFace buffalo_s pack)
curl -L -o models/w600k_mbf.onnx \
  https://huggingface.co/immich-app/buffalo_s/resolve/main/recognition/model.onnx
```

Note: the InsightFace model is distributed for research/educational use;
review its license before any commercial deployment.

### 6. Verify your setup

```bash
KEY=$(cat ~/.mo-creds)
EP=https://api.cache.cell-4-us-west-2-1.prod.a.momentohq.com   # your region's endpoint
curl -X PUT -H "Authorization: $KEY" --data "hello" \
  "$EP/cache/<cache>?key=smoke&ttl_seconds=60"        # expect: 204
curl -H "Authorization: $KEY" "$EP/cache/<cache>?key=smoke"    # expect: 200 with body "hello"
```

This checks the Momento half only. Once Docker is running you can hand
the agent the full Module 0 prompt, which also verifies Valkey:

> Verify my setup: Momento PUT/GET smoke test with the key in
> `~/.mo-creds` against cache `<cache>` at endpoint `<endpoint>`, and
> confirm my local valkey at port `<port>` has the search module loaded.
> Never echo the key.

### 7. Run the workshop

Open `SYLLABUS.md` and work through the modules in order, pasting each
prompt into your agent (`PROMPTS.md` collects them all in one sheet).
Every module ends with checks, so you always know your build works
before moving on.

| Module | What you build | Time |
|---|---|---|
| 0 | Setup: Momento smoke test + local Valkey | 20 min |
| 1 | Face pipeline + threshold calibration (native) | 60 min |
| 2 | Valkey embeddings index + KNN recognizer | 45 min |
| 3 | Live video recognition (ffmpeg frame tap) | 40 min |
| 4 | Momento as the HLS streaming origin (optional) | 45 min |
| 5 | Enrollment (multiple photos per person) | 15 min |
| 6 | Write the blog post | 45 min |
| 7 | Reflect: interrogate what you built | 30 min |
| Take-home | Hello-world wasm Momento Function | 30 min |

Core run (0-3, 5, 6): ~3.75 hours. Everything including the optional
streaming module and the reflection wrap-up: ~5 hours. Plan a day with
slack, or two half days split after Module 2.

**Short on time (2 hours)?** Do 0 and 1-2 compressed (the shipped
`faces/` and `faces-library.json` let you skip photo hunting and even
library building). Watch 3-4 as a demo; take 5-7 and the take-home home.

## Notes

- `faces/` contains photos of public figures for pipeline testing only;
  see `faces/README.md` for sources and terms. Swap in your own photos to
  make Module 5 (enrolling yourself) more fun.
- No credentials live in this repo, and none should ever be committed.
  Keys belong in `~/.mo-creds` (mode 600).
- Code in this repo is MIT licensed (`LICENSE`). The ML models and the
  face photos are NOT covered by that license; they keep their own terms.
