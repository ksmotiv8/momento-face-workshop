# Workshop face data

Test data for the recognition pipeline:

- `portraits/` - one photo per identity. Used to BUILD the face library
  (Module 2). Filenames become the identity names.
- `probes/` - photos used to TEST the deployed function (Module 3):
  second photos of library identities (genuine pairs), multi-face photos,
  and strangers who should stay unmatched.

## Sources and terms

These are photographs of public figures collected from publicly available
sources (public-domain government portraits and images circulated for
press/educational use, including samples from the widely used
ageitgey/face_recognition examples). They are included solely as test
data for an educational computer-vision exercise. All rights remain with
their respective owners; no endorsement by the pictured individuals is
implied. If you are an owner and want an image removed, open an issue.

These images are NOT covered by the repository's MIT license.

## Better: use your own

The workshop is more fun with faces you know. Replace or extend these
directories with your own photos (one clear, mostly frontal face per
portrait; a couple of different shots per person for probes) and the
exercises work unchanged - Module 6 lets you enroll yourself live.
