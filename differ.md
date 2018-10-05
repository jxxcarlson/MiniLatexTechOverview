# Differ

MiniLatex uses a number of optimaztions. One of these
is to re-parser only paragraphs which have. This is
relevant to the editing process, since instantaneous
feedback is essential for a satisfying user experience.

The strategy is as follows. Let `u` and `v` be two lists
of strings. Write them as `u = a x b`, `v = u y b`,
where `a` is the greatest common prefix and `b` is the
greatest common suffix. Supposed that we have in hand
`U = A X B`, where `A = render a`, etc. Let `Y = render y`.
Then `V = A Y B`.
