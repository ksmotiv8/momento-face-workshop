# What is vector search?

A model turns each thing (a face, a sentence, a product photo) into a
list of numbers called a vector. Our face model outputs 512 numbers per
face. The model was trained so that similar things get similar numbers.

That has a nice consequence: every face becomes a point in space, and
photos of the same person land close together.

```
   similarity space (2 of the 512 dimensions shown)

   |          Rose
   |          o  o
   |
   |                          Obama
   |    Biden                 o  o
   |    o  o                   o   x   <- query: a face from
   |     o                              a new video frame
   |
   +---------------------------------------
```

Vector search is just this question: given a new point, which stored
points are nearest? Here the query `x` sits inside the Obama cluster, so
the answer comes back "Barack Obama" with a distance that says how close.
A stranger's face lands far from every cluster, so the nearest match is
still a bad one, and a threshold rejects it.

"Near" needs a definition. This workshop uses cosine: how much two
vectors point in the same direction. One trap to know: some systems
report similarity (1.0 = identical) and others report distance
(0.0 = identical). Valkey reports distance. Mix them up and everything
matches, or nothing does.

With 5 people you can afford to compare the query to every stored vector.
With 50 million you cannot. That is what a vector index is for: HNSW (the
one Valkey builds in Module 2) keeps shortcut links between neighboring
points, so a query hops toward its own neighborhood and checks only a
tiny fraction of the data. The answer is approximate but close, and it
comes back in about a millisecond instead of a linear scan.

Where you touch all of this in the workshop: Module 1 turns faces into
vectors and picks a threshold, Module 2 stores them in Valkey and asks it
the nearest-neighbor question with `FT.SEARCH ... KNN`.
