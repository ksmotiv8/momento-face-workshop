# Workshop face data

Test data for the recognition pipeline:

- `portraits/` - photos used to BUILD the face library (Module 1).
  Filenames become the identity names: strip the extension, drop a
  trailing numeric suffix, replace underscores with spaces - so
  `Barack_Obama.jpg` AND `Barack_Obama_2.jpg` both label "Barack Obama"
  (multiple entries per person; nearest wins at match time). Apply that
  normalization in your builder or your names will not match the shipped
  library's.
- `probes/` - photos used to TEST recognition (Modules 1-2): second
  photos of library identities (genuine pairs), multi-face photos, and
  strangers who should stay unmatched. Note: Alex Lacamoire has no
  second photo here, so he contributes no genuine pair to calibration -
  add your own `alex2.jpg` or leave him out of genuine-pair stats.
- `faces-library.json` - a prebuilt embeddings library for the 5
  portraits (512-D vectors from the w600k_mbf model, L2-normalized,
  ~31 KB). Schema: a bare JSON array (no wrapper object) of entries
  `{"name": "First Last", "e": [512 floats]}` - names use spaces, per
  the filename normalization above. This is the CANONICAL library
  path for the whole workshop: Module 1's `build` writes here
  (overwriting this shipped copy; it is git-tracked, so
  `git diff` compares yours to ours and `git checkout` restores it),
  and Module 2's `index` reads from here. Skipping Module 1 works
  because this shipped copy is already in place. Caveat: embeddings only compare if your
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
genuine pairs, which is what makes the Module 1 threshold calibration
honest (same-person similarity measured across different photos). If the
extra photos joined the library, probing with them would test on training
data and every score would be a meaningless ~0.98.

Matching handles multiple entries per person - the nearest entry wins -
and production libraries should carry 2-3 photos per person for
pose/lighting robustness. Both paths support it: the Module 1 builder
via the `Name_2.jpg` suffix convention above, and Module 5's `enroll`
subcommand (`face:<name>:<n>` keys). Just keep genuine-pair probes OUT
of the library: a probe photo that also became a library entry scores a
meaningless ~1.0 against itself.

## Better: use your own

The workshop is more fun with faces you know - Exercise 0b in the
syllabus walks through photographing people around you (with their
permission), labeling the files, and dropping them here. One clear,
mostly frontal face per portrait; a different second shot per person
into `probes/` for genuine pairs. The exercises then work unchanged,
and Module 5 lets you enroll more people live.
