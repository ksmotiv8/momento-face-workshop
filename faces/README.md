# Workshop face data

Test data for the recognition pipeline:

- `portraits/` - one photo per identity. Used to BUILD the face library
  (Module 2). Filenames become the identity names: strip the extension
  and replace underscores with spaces (`Barack_Obama.jpg` -> the name
  "Barack Obama"). Apply that normalization in your builder or your
  names will not match the shipped library's.
- `probes/` - photos used to TEST the deployed function (Module 3):
  second photos of library identities (genuine pairs), multi-face photos,
  and strangers who should stay unmatched.
- `faces-library.json` - a prebuilt embeddings library for the 5
  portraits (512-D vectors from the w600k_mbf model, L2-normalized,
  ~31 KB). Schema: a bare JSON array (no wrapper object) of entries
  `{"name": "First Last", "e": [512 floats]}` - names use spaces, per
  the filename normalization above. Two uses: PUT it straight to your
  cache to unblock Module 3
  without finishing Module 2, or diff it against your own `build` output
  to verify your pipeline. Caveat: embeddings only compare if your
  preprocessing matches the skill's recipe exactly (RGB, +25% square
  crop, 112x112 Triangle resize, (x-127.5)/127.5). If your genuine-pair
  scores against this library look mysteriously low, your preprocessing
  diverged - that is itself a useful lesson.

```bash
# use it directly: PUT to your cache and the recognizer is live
KEY=$(cat ~/.mo-creds)
curl -X PUT -H "Authorization: $KEY" --data-binary @faces/faces-library.json \
  "https://<endpoint>/cache/<cache>?key=faces-library.json&ttl_seconds=86400"
```

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
(or a centroid) for pose/lighting robustness. To do that here you need
two small changes: a naming convention the library builder understands
(`Name_2.jpg` must map to "Name", not "Name 2"), and dropping the
one-vector-per-person replacement in the enrollment function. A good
stretch exercise after Module 6.

## Better: use your own

The workshop is more fun with faces you know. Replace or extend these
directories with your own photos (one clear, mostly frontal face per
portrait; a couple of different shots per person for probes) and the
exercises work unchanged - Module 6 lets you enroll yourself live.
