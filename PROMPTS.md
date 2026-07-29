# Prompt Sheet

The workshop prompts in one place, in order (matching `SYLLABUS.md`;
Module 7 lists the core reflect prompts, with the full set in the
syllabus).
Paste them into your coding agent one at a time; read what comes back
before moving on. Replace `<cache>` and `<endpoint>` with your values. Other angle-bracket tokens inside prompts (like `<name>` in the
greeting or enroll exercises) are program inputs - leave them to the
agent.

Two habits that make these work:

- End asks with "verify it works" or "prove it" - an agent that must
  demonstrate success debugs itself.
- When something looks wrong, describe the symptom, not your theory:
  "the video is racing" beats "I think the timestamps are wrong."

## Module 0 - Setup

> Verify my setup: Momento PUT/GET smoke test with the key in
> `~/.mo-creds` against cache `<cache>` at endpoint `<endpoint>`, and
> confirm my local valkey at port 6379 has the search module loaded.
> Never echo the key.

Optional (after photographing your crew into faces/, see Exercise 0b):

> I added new photos to `faces/portraits` and `faces/probes`. Check every
> new file: valid image, filename in `First_Last.jpg` form, and no
> duplicate names. List what will become each person's label. Face
> quality gets checked for real in Module 1's build - flag anything that
> looks likely to fail (tiny file, screenshot, group shot).

## Module 1 - Face pipeline, native

> Set up a Rust workspace with a shared `facecore` crate
> (detect -> crop 112x112 chip -> embed -> L2-normalize -> cosine match)
> and a host CLI `libbuild` with subcommands `build` (portraits dir ->
> library JSON), `probe` (score images against the library) and `pairs`
> (all-pairs cosines). Labels come from filenames: strip the extension,
> drop a trailing numeric suffix, replace underscores with spaces - so
> `Barack_Obama.jpg` and `Barack_Obama_2.jpg` are BOTH labeled 'Barack
> Obama' (multiple library entries per person is supported; nearest
> wins). Use rustface for detection and tract-onnx for the ArcFace
> embedder. Verify it works end to end on one portrait before building
> out the rest.

> Build a library from the portraits in `faces/portraits`, then measure
> impostor cosines (`pairs`) and genuine cosines (probe the second photos
> in `faces/probes`). Report both ranges and recommend a threshold in the
> gap.

## Module 2 - Valkey as the embeddings index

> Write a `libbuild index` subcommand that loads `faces-library.json`
> into my local valkey as an HNSW COSINE index (FT.CREATE, one HSET per
> entry with the vector packed as little-endian f32 plus a name field).
> Then verify with an FT.SEARCH KNN query using one of the library
> vectors - it must return its own name at distance near zero.

> Write a `recognize` binary: given an image, detect faces with facecore,
> embed each, and match via FT.SEARCH KNN against valkey instead of brute
> force. Careful: valkey returns cosine DISTANCE (1 - similarity), so my
> similarity threshold of T accepts matches with dist <= 1 - T. Run the
> full probe set from `faces/probes` and report a table: genuine pairs
> named, multi-face photos counted correctly, strangers unmatched.

## Module 3 - Live video, local recognition

> Build a simulated camera stream with ffmpeg: `testsrc2` background with
> two known face images overlaid on co-prime periods (so they appear
> alone, together, and not at all). Size the overlays so each face clears
> the 20 px detection floor at the tap resolution. Split it: one branch
> encodes H.264, the other taps 2 fps to an atomically-rewritten JPEG.
> Verify the injected faces still recognize as stills before wiring up
> video.

> Wire the stream tap to my local recognizer: watch the tapped JPEG's
> mtime, run `recognize` on each new frame, and log capture time,
> per-frame latency, and names. Confirm the log shows the
> alone/together/neither cases from the live stream. Then PUT each result
> as JSON to my Momento cache under `faces.json` (ttl 60) so a remote
> dashboard could poll it. Report the latency distribution.

Optional, needs a webcam:

> Swap the simulated stream for my real camera: same split, same 2 fps
> tap, same sidecar, but the input is my webcam (avfoundation on macOS,
> v4l2 on Linux - the ffmpeg skill has the exact flags). Show me the
> sidecar log while I sit in front of the camera.

Optional feedback loop (Topics):

> Extend the sidecar: when a NEW person appears (a name not present in
> the previous frame - transitions only, not every frame), publish a
> small JSON event to Momento topic `face-events` via
> POST `/topics/<cache>/face-events`. Then show me feedback two ways:
> (1) `momento topic subscribe` in a second terminal printing events
> live; (2) pipe each arriving event into text-to-speech (`say` on
> macOS, `espeak` on Linux) so the room hears 'Welcome, <name>' when
> someone steps in front of the camera.

## Module 4 - Momento as streaming origin (optional)

> Publish my ffmpeg stream as HLS directly to Momento as origin: 1 s
> segments PUT to cache keys with `-method PUT -http_persistent 1`, the
> playlist as another key. Cap the bitrate so segments stay under the
> 1 MiB item limit, set `-g` = fps so every segment starts on a keyframe,
> and put `&token=` in the segment filename template so players can
> resolve relative URIs. Verify by fetching the playlist and watching
> MEDIA-SEQUENCE advance, then fetch a segment and check the first byte
> is 0x47. For the `&token=` in the playlist, use a scoped short-lived
> key minted for this stream, not my main workshop key.

Contingency (not part of the prompt): if ffmpeg reports "Failed to
resolve hostname" while curl to the same host works, the ffmpeg build's
resolver is broken (common in static builds) - use the curl-loop
fallback in the momento-streaming-origin skill.

## Module 5 - Enrollment

> Add a `libbuild enroll <image> <name>` subcommand: detect the largest face,
> embed it, HSET it into the valkey index, and prove the very next
> `recognize` run names that person. Support multiple entries per person
> (face:<name>:<n> keys) - nearest entry wins.

## Module 6 - The blog post

> Write a blog post about what we built in this workshop, for an engineer
> who has never used Momento or a vector index. Pull every number from
> THIS session's real measurements - my measured impostor/genuine cosine
> ranges and threshold, KNN distances, per-frame latency - not from the
> syllabus. Structure: the hook (what runs where, and what is NOT
> running), the architecture in one diagram, how recognition actually
> works (embeddings, not training), a local-vs-serverless section (use
> the syllabus Reference numbers for the serverless side, or your own if
> you built the function-based recognizer from the face-recognition skill), three things that went wrong and what each
> taught us, and what we would build next. Include real commands and
> responses where they carry the story. No em-dashes. Do not brag about
> platform internals - write what a reader can use.

## Module 7 - Reflect

Run these one at a time; argue with the answers. The full set with
context is in `SYLLABUS.md` Module 7.

> Walk me through every design decision we made that I did not explicitly
> ask for. For each: what was the alternative, and why did you pick this?

> Why do embeddings work for recognition at all? Explain what the 512
> numbers mean, why cosine similarity is the right comparison, and what
> the L2 normalization is for - to someone who knows no ML.

> What breaks first if 100 cameras hit this system at once? Walk the
> request path and find the bottleneck.

> Quiz me: 10 questions about what we built, one at a time, hardest
> first. Wait for my answer before showing yours. Keep score honestly.

> Our ML runs natively on this machine. Walk me through what would
> change - and what would not - if the recognizer moved inside a Momento
> Function as a shared service. What does that say about where the real
> complexity in this system lives?

> Explain why valkey returns distance while our threshold was calibrated
> as similarity, and what would have happened if we had used 0.30 as a
> distance cutoff. What is the general lesson about porting numbers
> between systems?

## Take-home - Hello World Momento Function

Needs `rustup target add wasm32-wasip2` and the `momento-functions`
skill loaded.

> Create a Rust Momento Function that returns 'hello world' as a plain
> string. Build for wasm32-wasip2, deploy to cache `<cache>` as `hello`,
> and invoke it to prove it works. Put the deploy+invoke steps in a
> reusable script that reads the key from `~/.mo-creds` inside the script.

> Change the function to accept `{"name":"..."}` and return
> `{"message":"Hello <name>"}` using `Json<T>` for request and response.
> Redeploy and test with my name.

## When things go sideways

Real recovery prompts. None contain a diagnosis - that is the agent's
job:

> The video is playing way too fast.

> The sidecar says zero faces on every frame, but stills recognize fine.

> ffmpeg says it cannot resolve the hostname, but curl to the same host
> works.

> Everything suddenly matches everyone. What changed?
