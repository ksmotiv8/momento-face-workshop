---
name: ffmpeg
description: General ffmpeg skill - command anatomy, everyday conversions (transcode, trim, concat, resize, extract audio), capturing cameras/screens/mics per OS, live streaming and HLS origin settings, drawing text and image overlays, generating and handling sound, extracting or sending frames out of video, and debugging playback/timing problems. Use for any ffmpeg, video, or audio processing task.
---

# ffmpeg

> **Guard - reference only. Loading this skill is not a request to act.**
> Do not create, build, deploy, or run anything based on this skill unless
> the user has explicitly asked for that work in this conversation. If they
> have not asked, answer from it and stop.

## Command anatomy (read this first)

An ffmpeg command is a graph: inputs -> filters -> encoders -> muxers ->
outputs. Argument POSITION carries meaning:

- Options before an `-i` apply to that input (`-f avfoundation -framerate 30 -i "0"`).
- Options after the inputs and before an output URL apply to that output
  (codecs, bitrate, `-map`, `-vf`).
- Multiple outputs: repeat `[output options] <url>` blocks; each output
  encodes independently. `-map` picks which streams go to which output.
- `-vf`/`-af` build a simple per-output filter chain. `-filter_complex`
  builds one shared graph with named pads (`[v1]`, `[out]`) that outputs
  reference via `-map "[out]"`.
- `-c copy` remuxes without re-encoding: instant, lossless, but cuts land
  on keyframes only.
- One process can do several jobs at once: stream live, record to disk,
  and emit analysis frames from a single capture.

Check the build before relying on filters: `ffmpeg -filters | grep drawtext`.
Static builds often lack libfreetype (drawtext) and some capture devices;
prefer distro or brew builds when drawing text.

## Everyday operations

```bash
ffmpeg -i in.mov out.mp4                                   # transcode (defaults are sane)
ffmpeg -i in.mp4 -c copy out.mkv                           # remux, no re-encode
ffmpeg -ss 00:01:30 -to 00:02:00 -i in.mp4 -c copy cut.mp4 # trim (fast, keyframe-aligned)
ffmpeg -ss 90 -i in.mp4 -t 30 -c:v libx264 cut.mp4         # trim (re-encode, frame-exact)
ffmpeg -f concat -safe 0 -i list.txt -c copy joined.mp4    # concat (list.txt: file 'a.mp4' lines)
ffmpeg -i in.mp4 -vf scale=1280:-2 out.mp4                 # resize (-2 keeps even height)
ffmpeg -i in.mp4 -vn -c:a libmp3lame audio.mp3             # extract audio
ffmpeg -i in.mp4 -an muted.mp4                             # drop audio
ffmpeg -i in.mp4 -vf "fps=12,scale=480:-1" out.gif         # gif
ffmpeg -i in.mp4 -filter:v "setpts=0.5*PTS" fast.mp4       # 2x speed (video)
ffmpeg -framerate 30 -i img%04d.png -pix_fmt yuv420p out.mp4  # images -> video
```

Web-safe defaults: `-c:v libx264 -pix_fmt yuv420p -c:a aac -movflags +faststart`.

## Capturing (cameras, screens, mics)

- macOS video: `-f avfoundation -framerate 30 -video_size 1280x720 -i "0"`
  (list devices: `ffmpeg -f avfoundation -list_devices true -i ""`).
  Add `-pix_fmt yuv420p` on the encode: avfoundation emits 4:2:2, which
  browsers cannot decode. Screen is a device index too (`-i "1"`).
- Linux video: `-f v4l2 -framerate 30 -video_size 1280x720 -i /dev/video0`.
  Devices are group `video` (`ls -l /dev/video0`). In Docker the device
  must be mapped in; mapping a missing device fails container creation
  rather than falling back. X11 screen: `-f x11grab -i :0`.
- Mics: macOS `-f avfoundation -i ":0"` (video:audio indexes, `"0:0"` for
  both); Linux `-f alsa -i default` or `-f pulse -i default`.
- No hardware / testing: `-f lavfi -i testsrc2=size=1280x720:rate=30`.
  Add `-re` to pace synthetic sources at realtime for live use; without it
  lavfi renders as fast as the CPU allows.

## Sound

- Generate a tone: `-f lavfi -i sine=frequency=440:sample_rate=44100`.
- Silent track: `-f lavfi -i anullsrc=channel_layout=stereo:sample_rate=44100`.
  Add one when capturing video-only: players behave better with an audio
  track present, and on macOS video+anullsrc means a camera permission
  prompt but no microphone prompt.
- Encode `-c:a aac -b:a 128k` for HLS/web; map explicitly with multiple
  inputs: `-map 0:v -map 1:a`.
- Mix two sources: `-filter_complex amix=inputs=2`.
- Timing hazard: NEVER use `-use_wallclock_as_timestamps` with unpaced
  lavfi audio. Each audio packet gets stamped "now", the audio timeline
  collapses, and players (which slave video to the audio clock) race the
  video at superspeed. The flag does not deliver absolute wall-clock PTS
  anyway (inputs get rebased). For A/V drift, use
  `-af aresample=async=1000` instead.

## Live streaming and HLS origin settings

Low-latency encode baseline:

```
-c:v libx264 -preset veryfast -tune zerolatency -g <fps*seg_seconds> -sc_threshold 0 \
-pix_fmt yuv420p -c:a aac -b:a 128k
```

- `-g` (keyframe interval) must equal fps x segment seconds so every
  segment starts on a keyframe; `-sc_threshold 0` stops scene-cut
  keyframes from breaking the cadence. Wrong cadence = segment durations
  drift from `hls_time`.
- Rate-cap when the origin limits object size:
  `-b:v 1500k -maxrate 1800k -bufsize 3600k` keeps 1s 720p segments ~200 KB.

HLS muxer:

```
-f hls -hls_time 1 -hls_list_size 6 -hls_flags program_date_time
```

- `hls_time`: latency scales with it. Plain-HLS glass-to-glass bottoms out
  near 2x segment duration plus player buffer; 1s segments with a player
  synced 2 segments from the live edge gives ~3-5s.
- `hls_list_size`: rolling playlist window.
- `program_date_time` writes EXT-X-PROGRAM-DATE-TIME per segment: required
  to time-align external metadata with playback and to measure latency;
  sub-second accurate with short segments.
- Local disk target: add `delete_segments+append_list` to `hls_flags`;
  serve the directory with correct MIME types
  (`application/vnd.apple.mpegurl` for .m3u8, `video/mp2t` for .ts) and
  no-cache on the playlist.
- If ANYTHING watches the segment directory and ships files elsewhere (an
  uploader sidecar, a sync loop), also add `+temp_file` to `hls_flags`:
  segments then appear atomically via rename. Without it, ffmpeg writes
  segments in place and a watcher that uploads on first sight races the
  encoder and ships truncated .ts files that never get re-sent.
- HTTP origin target (PUT segments to object/cache storage):

```
-method PUT -http_persistent 1 \
-hls_segment_filename "https://origin/path/seg%05d.ts?<params>" \
"https://origin/path/stream.m3u8?<params>"
```

  `-http_persistent 1` matters: the connection-per-request path is flaky
  against some HTTPS origins (mid-upload TLS failures, sometimes silent).
  Auth headers go in `-headers "Name: value"` with a trailing CRLF. If the
  origin expires objects (TTLs), skip delete_segments.
- RTMP instead: `-f flv rtmp://server/app/key`.
- Debug rule: when uploads fail, reproduce the same PUT with curl before
  blaming the origin.
- Playback checks: `ffplay <url>`, or curl the playlist and watch
  `#EXT-X-MEDIA-SEQUENCE` advance once per segment duration.

## Drawing on video

Text:

```
drawtext=fontfile=/path/font.ttf:text='%{localtime\:%F %T} UTC':x=40:y=40:\
fontsize=56:fontcolor=white:box=1:boxcolor=black@0.6:boxborderw=12
```

- Escape colons inside `%{...}` as `\:`. Useful expansions: `%{localtime}`
  (wall clock), `%{pts:hms}` (stream time), `%{n}` (frame number).
- A burned-in wall clock is ground truth for latency and clock-skew
  measurement: compare against a live clock on screen or against
  PROGRAM-DATE-TIME.
- Fonts: alpine `ttf-dejavu` -> `/usr/share/fonts/dejavu/DejaVuSans-Bold.ttf`;
  macOS `/System/Library/Fonts/Supplemental/Arial.ttf`. Avoid `[]` in font
  paths (filtergraph syntax); copy the file somewhere plain if needed.

Image overlays (watermarks, picture-in-picture, injected content):

```
-loop 1 -framerate 5 -i logo.png ... \
-filter_complex "[1:v]scale=420:-1[ovl];[0:v][ovl]overlay=x=W-w-40:y=40[out]"
```

- `-loop 1` turns a still into an endless input; low `-framerate` keeps it
  cheap.
- Timed appearance: `overlay=enable='lt(mod(t,19),5)'` shows 5s out of
  every 19. Multiple overlays on co-prime periods produce varying
  combinations.
- Pseudo-random per-cycle position (filter expressions have no stateful
  random): `x='(main_w-overlay_w)*(0.5+0.5*sin(floor(t/19)*12.9898))'`.
- If overlaid content must be machine-detected downstream, use tight crops
  of the subject; a loose image scaled into a small analysis frame can sit
  below detector thresholds while looking obvious to humans.
- Graph order matters: overlays go UPSTREAM of a `split` when every output
  must see them; per-output touches (timestamps, scaling) go after the
  split on that branch.

## Getting images out of video

One-off extraction:

```bash
ffmpeg -i in.mp4 -frames:v 1 frame.png              # first frame
ffmpeg -ss 12 -i in.mp4 -frames:v 1 out.jpg         # frame at t=12s
ffmpeg -i in.mp4 -vf fps=1 thumbs/%04d.jpg          # 1 thumbnail per second
ffmpeg -i in.jpg -vf "crop=W:H:X:Y,scale=320:-1" out.jpg   # crop/resize a still
```

Continuous frame tap (feed an analysis service while streaming):

```
-filter_complex "...;[v]split[main][tap];[tap]fps=2,scale=256:-2[frames]" \
-map "[main]" ... <stream output> \
-map "[frames]" -q:v 4 -f image2 -update 1 -atomic_writing 1 /tmp/latest.jpg
```

- `fps=N` sets the tap rate; `scale=256:-2` keeps frames tiny (downstream
  decode cost scales with pixels, not JPEG bytes).
- `image2 -update 1 -atomic_writing 1` rewrites ONE file atomically
  (temp + rename); readers never see a partial JPEG.
- The file's mtime IS the frame's capture time (rename preserves it).
  Read it with sub-second precision portably (GNU `stat -c %.Y` exists,
  macOS stat does not; Python works everywhere):
  `python3 -c 'import os,sys;print(os.stat(sys.argv[1]).st_mtime_ns//1_000_000)' f.jpg`
  and forward it (e.g. an `x-frame-ts` header) so consumers can
  time-align results with the video.
- Ship frames with a sidecar loop polling mtime at 2x the tap rate:

```bash
mt(){ python3 -c 'import os,sys;print(os.stat(sys.argv[1]).st_mtime_ns//1_000_000)' "$1" 2>/dev/null; }
while sleep 0.25; do
  ms=$(mt /tmp/latest.jpg) || continue
  [ "$ms" = "$last" ] && continue; last=$ms
  curl -s -X POST -H "x-frame-ts: $ms" --data-binary @/tmp/latest.jpg "$SINK_URL"
done
```

- Alternatives: numbered files `-f image2 frames/%06d.jpg` (`-frame_pts 1`
  names by PTS); stdout stream `-f image2pipe -c:v mjpeg -`.
- Validate analysis pipelines offline first: compose ONE frame with the
  same filters (`-frames:v 1`) and feed it to the analyzer before touching
  the live pipeline.

## Debugging

```bash
ffprobe -hide_banner <input>              # streams, codecs, durations, timestamps
ffplay <url-or-file>                      # fastest playback check
ffmpeg -f lavfi -i testsrc2 -t 5 out.mp4  # known-good synthetic input
ffmpeg -v debug ... 2>&1 | grep -i http   # watch HTTP requests
```

Symptom map:
- Video races ahead of audio or plays superspeed: audio timeline broken
  (see Sound; usually wallclock timestamps or an unpaced source).
- Segments longer than `hls_time`: keyframe cadence wrong (`-g`,
  `-sc_threshold 0`).
- Black video in browsers: pixel format (need yuv420p) or wrong MIME types.
- Uploads silently missing at the origin: reproduce with curl; try
  `-http_persistent 1`.
- Filter not found: static build; use a full build.
- "Non-monotonic DTS" warnings: input timestamp jitter; smooth audio with
  `aresample=async`, or ignore if ticks are tiny and playback is clean.
- Players glitch on segments shipped by a directory-watching uploader:
  partial segments; add `-hls_flags +temp_file`.
- Command ignores `-t` and never stops: two causes. (1) `-t` after the
  last output is silently ignored (the "Trailing option(s)" warning hides
  below -loglevel error); as an output option it must precede each
  output. (2) `-loop 1` image inputs are INFINITE and keep ffmpeg alive
  even after the main input's `-t` expires. Reliable forms: `-t` on
  every output, or `-t` on every input including the looped ones.
