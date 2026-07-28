# Workshop face data

Test data for the recognition pipeline:

- `portraits/` - one photo per identity. Used to BUILD the face library
  (Module 2). Filenames become the identity names: strip the extension
  and replace underscores with spaces (`Barack_Obama.jpg` -> the name
  "Barack Obama"). Apply that normalization in your builder or your
  names will not match the shipped library's.
- `probes/` - photos used to TEST recognition (Modules 2-3): second
  photos of library identities (genuine pairs), multi-face photos, and
  strangers who should stay unmatched. Note: Alex Lacamoire has no
  second photo here, so he contributes no genuine pair to calibration -
  add your own `alex2.jpg` or leave him out of genuine-pair stats.
- `faces-library.json` - a prebuilt embeddings library for the 5
  portraits (512-D vectors from the w600k_mbf model, L2-normalized,
  ~31 KB). Schema: a bare JSON array (no wrapper object) of entries
  `{"name": "First Last", "e": [512 floats]}` - names use spaces, per
  the filename normalization above. Two uses: feed it straight to
  `libbuild index` (Module 3) to load Valkey without finishing
  Module 2's library build, or diff it against your own `build` output
  to verify your pipeline. Caveat: embeddings only compare if your
  preprocessing matches the skill's recipe exactly (RGB, +25% square
  crop, 112x112 Triangle resize, (x-127.5)/127.5). If your genuine-pair
  scores against this library look mysteriously low, your preprocessing
  diverged - that is itself a useful lesson.

## Sources and terms

These are photographs of public figures collected from publicly available
sources (public-domain government portraits and images circulated for
press/educational use, including samples from the widely used
ageitgey/face_recognition examples). They are included solely as test
data for an educational computer-vision exercise. All rights remain with
their respective owners; no endorsement by the pictured individuals is
implied. If you are an owner and want an image removed, open an issue.

These images are NOT covered by the repository's MIT license.

## Why only one portrait per identity?

It is a train/test split, not a limit of the pipeline. `portraits/`
becomes the library; the second photos live in `probes/` as HELD-OUT
genuine pairs, which is what makes the Module 2 threshold calibration
honest (same-person similarity measured across different photos). If the
extra photos joined the library, probing with them would test on training
data and every score would be a meaningless ~0.98.

Matching itself handles multiple entries per person fine - the nearest
entry wins - and production libraries should carry 2-3 photos per person
(or a centroid) for pose/lighting robustness. Module 6's `enroll`
subcommand already supports this (`face:<name>:<n>` keys). The one gap
left as a stretch exercise: teach the Module 2 library BUILDER a naming
convention so `Name_2.jpg` maps to "Name" rather than a new person
called "Name 2".

## Better: use your own

The workshop is more fun with faces you know. Replace or extend these
directories with your own photos (one clear, mostly frontal face per
portrait; a couple of different shots per person for probes) and the
exercises work unchanged - Module 6 lets you enroll yourself live.
