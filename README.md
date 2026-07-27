# Momento Face Recognition Workshop

Build a serverless face recognition service with a coding agent: WebAssembly
functions running next to a Momento cache detect and recognize faces, the
face library lives in the cache (add people with no redeploy), and a live
video stream gets recognized in flight with Momento as the HLS origin.

```
camera / test stream ──ffmpeg──► Momento cache "origin"  ◄──── browser / players
                        │              ▲
                        └─frames──► Momento Function (wasm)
                                    detect -> embed -> match -> names
```

You drive the build by prompting a coding agent. This repo gives the agent
the knowledge it needs (the `skills/`), gives you the exercises
(`SYLLABUS.md`, `PROMPTS.md`), and gives the pipeline test data (`faces/`).

## Repo layout

```
SYLLABUS.md          the workshop: 8 modules with prompts, timings, expected results
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

The cache must exist before anything else works: functions deploy onto a
cache, and all data lives at `/cache/<cache>?key=...`.

### 2. Install the tools

```bash
# Rust + the wasm target (functions compile to wasm32-wasip2)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32-wasip2

# ffmpeg (modules 4-5) - use a full build, not a static one
brew install ffmpeg            # macOS
sudo apt install ffmpeg        # Debian/Ubuntu

# Momento CLI (optional but handy: cache create/list/set/get)
brew install momentohq/tap/momento-cli          # macOS
# or grab a release: https://github.com/momentohq/momento-cli/releases
```

### 3. Install a coding agent

Either works; the skills are compatible with both.

- **mo** (Momento's native coding agent): get the binary from your Momento
  contact, put it on your PATH, then `mo login` (browser OAuth). mo reads
  project skills from `.mo/skills/` and also honors Claude Code skills.
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
platform behavior (deploy payload shapes, memory limits, HLS quirks) the
hard way.

### 5. Download the models

The two ML models are not in this repo (size and licensing). Download them
once into the crate directory the agent creates in Module 2:

```bash
# SeetaFace frontal detector (~1.2 MB)
curl -L -o seeta_fd_frontal_v1.0.bin \
  https://github.com/atomashpolskiy/rustface/raw/master/model/seeta_fd_frontal_v1.0.bin

# w600k_mbf ArcFace embedder (~13.6 MB, InsightFace buffalo_s pack)
curl -L -o w600k_mbf.onnx \
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
curl -H "Authorization: $KEY" "$EP/cache/<cache>?key=smoke"    # expect: hello
```

Or hand it to the agent (the Module 0 prompt):

> Verify my Momento setup: my API key is in `~/.mo-creds` and my cache is
> `<name>` at endpoint `<endpoint>`. Do a PUT then GET of a test key and
> confirm both succeed. Never echo the key.

### 7. Run the workshop

Open `SYLLABUS.md` and work through the modules in order, pasting each
prompt into your agent. `PROMPTS.md` has all the prompts in one place.
Every module ends with checks, so you always know your build works before
moving on.

| Module | What you build | Time |
|---|---|---|
| 0 | Setup smoke test | 15 min |
| 1 | Hello-world wasm function | 30 min |
| 2 | Face pipeline + threshold calibration (offline) | 60 min |
| 3 | Deploy the recognizer + test ladder | 45 min |
| 4 | Live video recognition (ffmpeg frame tap) | 45 min |
| 5 | Momento as the HLS streaming origin | 45 min |
| 6 | Enrollment with no redeploy | 30 min |
| 7 | Write the blog post | 45 min |

Full run: ~5.25 hours nominal; plan a day, or two half days split after
Module 3.

**Short on time (2 hours)?** Do 0, 1, 2 (with the provided `faces/`
instead of hunting photos), and 3. Watch 4-5 as a demo; take 6-7 home.

**Before a group workshop:** ask Momento support to raise the function
memory limit on the workshop account. The face function embeds a 13.6 MB
model and exceeds the default budget; Module 6 blocks on this.

## Notes

- `faces/` contains photos of public figures for pipeline testing only;
  see `faces/README.md` for sources and terms. Swap in your own photos to
  make Module 6 (enrolling yourself) more fun.
- No credentials live in this repo, and none should ever be committed.
  Keys belong in `~/.mo-creds` (mode 600).
- Code in this repo is MIT licensed (`LICENSE`). The ML models and the
  face photos are NOT covered by that license; they keep their own terms.
