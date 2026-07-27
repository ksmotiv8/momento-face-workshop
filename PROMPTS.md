# Prompt Sheet

Every workshop prompt in one place, in order. Paste them into your coding
agent one at a time; read what comes back before moving on. Replace
`<cache>`, `<endpoint>`, and `<name>` with your values.

Two habits that make these work:

- End asks with "verify it works" or "prove it" - an agent that must
  demonstrate success debugs itself.
- When something looks wrong, describe the symptom, not your theory:
  "the video is racing" beats "I think the timestamps are wrong."

## Module 0 - Setup

> Verify my Momento setup: my API key is in `~/.mo-creds` and my cache is
> `<cache>` at endpoint `<endpoint>`. Do a PUT then GET of a test key and
> confirm both succeed. Never echo the key.

## Module 1 - Hello world function

> Create a Rust Momento Function that returns 'hello world' as a plain
> string. Build for wasm32-wasip2, deploy to cache `<cache>` as `hello`,
> and invoke it to prove it works. Put the deploy+invoke steps in a
> reusable script that reads the key from `~/.mo-creds` inside the script.

> Change the function to accept `{"name":"..."}` and return
> `{"message":"Hello <name>"}` using `Json<T>` for request and response.
> Redeploy and test with my name.

## Module 2 - Face pipeline, offline

> Set up a Rust workspace with a shared `facecore` crate
> (detect -> crop 112x112 chip -> embed -> L2-normalize -> cosine match)
> and a host CLI `libbuild` with subcommands `build` (portraits dir ->
> library JSON), `probe` (score images against the library) and `pairs`
> (all-pairs cosines). Use rustface for detection and tract-onnx for the
> ArcFace embedder. Verify it works natively before any wasm work.

> Build a library from the portraits in `faces/portraits`, then measure
> impostor cosines (`pairs`) and genuine cosines (probe the second photos
> in `faces/probes`). Report both ranges and recommend a threshold in the
> gap.

## Module 3 - Deploy the recognizer

> Write a Momento Function that accepts a JPEG POST body and returns
> `{"faces":[{x,y,w,h,score,name,sim}]}`. Embed both models with
> `include_bytes!`, read the library from cache key `faces-library.json`
> on each invoke, and cache the parsed ONNX plan in a `OnceLock`. Bad
> input must return a clean 400, never a panic. Deploy it and PUT the
> library to cache.

> Test my deployed face function in this order and report a table:
> (1) garbage bytes - expect a clean 400, not a 500; (2) a library
> portrait - expect its own name near 0.98; (3) a different photo of the
> same person; (4) a multi-face photo; (5) strangers - expect faces with
> no name; (6) the same portrait 10x - confirm warm latency is stable.

> Probe my function with a group photo containing many small faces. If it
> finds none, diagnose why by computing the face size after downscaling to
> the detection plane, then raise the plane via the `DETECT_PLANE` env var
> and re-test. Report faces found and latency at both settings.

## Module 4 - Live video recognition

> Build a simulated camera stream with ffmpeg: `testsrc2` background with
> two known face images overlaid on co-prime periods (so they appear
> alone, together, and not at all). Split it: one branch encodes H.264,
> the other taps 2 fps to an atomically-rewritten JPEG. Verify the
> injected faces still recognize as stills before wiring up video.

> Write a sidecar that polls the tapped JPEG's mtime at 2x the tap rate,
> POSTs each new frame to my face function, and logs capture time, frame
> age, RTT and recognized names. Run ~24 frames and report latency broken
> down by number of faces in frame.

## Module 5 - Momento as streaming origin

> Publish my ffmpeg stream as HLS directly to Momento as origin: 1 s
> segments PUT to cache keys with `-method PUT -http_persistent 1`, the
> playlist as another key. Cap the bitrate so segments stay under the
> 1 MiB item limit, set `-g` = fps so every segment starts on a keyframe,
> and put `&token=` in the segment filename template so players can
> resolve relative URIs. Verify by fetching the playlist and watching
> MEDIA-SEQUENCE advance, then fetch a segment and check the first byte
> is 0x47.

## Module 6 - Enrollment without redeploy

> Write a second Momento Function `add-face` that takes a JPEG body plus
> `?name=<person>`, detects the largest face, embeds it, and appends it to
> `faces-library.json` in the cache. Then confirm my face-detect function
> recognizes the new person on the very next invoke, with no redeploy.

## Module 7 - The blog post

> Write a blog post about what we built in this workshop, for an engineer
> who has never used Momento. Pull every number from THIS session's real
> measurements - deploy times, warm vs cold invoke latency, my measured
> impostor/genuine cosine ranges and threshold, per-face latency, segment
> sizes - not from the syllabus. Structure: the hook (what runs where, and
> what is NOT running), the architecture in one diagram, how recognition
> actually works (embeddings, not training), three things that went wrong
> and what each taught us, and what we would build next. Include real
> commands and responses where they carry the story. No em-dashes. Do not
> brag about platform internals - write what a reader can use.

## When things go sideways

Real recovery prompts from building this. None contain a diagnosis - that
is the agent's job:

> The video is playing way too fast.

> I reloaded and the video isn't playing.

> The timing between the boxes and the faces feels off. Who draws the
> boxes, and can we make it frame-accurate?

> Every invoke returns a bare 500 with no message. Figure out where it
> dies without logs.
