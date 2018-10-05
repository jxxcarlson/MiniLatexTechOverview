# Differ

MiniLatex uses a number of optimizations. One of these
is to re-parse and re-render only paragraphs which have
changed. This is optimization is important for the editing
process, since instantaneous feedback is essential for a
satisfying user experience. Such differential transformations
are more important for parsing than for rendering, since
parsing is by far the most expensive process in terms
of execution time.

The strategy is as follows. Let `u` and `v` be two lists
of strings. Write them as `u = a x b`, `v = u y b`,
where `a` is the greatest common prefix and `b` is the
greatest common suffix. Supposed that we have in hand
`u' = render u = a' x' b'`, where `a' = render a`, etc.
Let `y' = render y`. Then `v' = diff u = a' y' b'`.

The strategy just outlined is far from optimal for arbitrary
changes to the list `v`. However, a real human editor generally
makes small, localized changes to `v`, so in "real life," the
strategy is near-optimal.

NOTE. I would like to thank Ilias van Peer, who
recommended using a diffing strategy for
performance optimization.

## Implementing the diff stategy

The stuctures `u = a x b`, `v = u y b` map directly
onto the `DiffRecord` type:

```haskell
type alias DiffRecord =
    { commonInitialSegment : List String
    , commonTerminalSegment : List String
    , middleSegmentInSource : List String
    , middleSegmentInTarget : List String
    }
```

The implementation of

```elm
diff : List String -> List String -> DiffRecord
```

is straightforward.

## Edit records

The function `diff` carries out the transformation `a x b => a y b`.
To effect the transformation `a' x' b' => a' y' b'`, one needs the notion of an
`EditRecord`, which carries the needed state:

```elm
type alias EditRecord a =
    { paragraphs : List String
    , renderedParagraphs : List a
    , latexState : LatexState
    , idList : List String
    }
```

The rendering function applied to an `EditRecord`
takes `paragraphs` and `LatexState` as arguments
and returns `renderedParagraphs`. The `idList`
is a list of `ids`, one for each rendered paragraph.
When an updated `EditRecord` is formed, the `ids`
of the changed paragraphs are changed. This is done
in order to help both the Elm renderer and MathJax
to be as efficient as possible. Again: speed optimizations.
The initial `ids` are like this `p.0.1`, `p.0.2`, `p.0.3`,
etc. That is, they are sequential in the third component.
Suppose that the paragraphs with `ids` `p.0.17` and `p.0.18`
are changed. The corresponding `ids` in the new rendered
text are something like `p.3785.17` and `p.3785.18`, where
`3785` is randomly generated.

## Differential rendering

Below is the definition of `differentialRender`. It takes
a rendering function, a `DiffRecord`, and an `EditRecord`
as input and produces a list of rendered paragraphs as output.

```elm
Differ.differentialRender : (String -> a) -> DiffRecord -> EditRecord a -> List a
Differ.differentialRender renderer diffRecord editRecord =
    let
        ii =
            List.length diffRecord.commonInitialSegment

        it =
            List.length diffRecord.commonTerminalSegment

        initialSegmentRendered =
            List.take ii editRecord.renderedParagraphs

        terminalSegmentRendered =
            takeLast it editRecord.renderedParagraphs

        middleSegmentRendered =
            List.map renderer diffRecord.middleSegmentInTarget
    in
    initialSegmentRendered ++ middleSegmentRendered ++ terminalSegmentRendered
```

## Updating an EditRecord

Updating an `EditRecord` is a multistep process. The `Differ.update`
function takes as input a random number seed, a rendering function,
and `EditRecord`, and a string of source text. The output is
an updated `EditRecord`. Here are the steps:

1. Apply `logicalParagraphify` to transform the input source text
   string into a list of logical paragraphs.

2. Apply `diff` to the given `EditRecord` and the list of logical paragraphs
   to produce a `DiffRecord`.

3. Apply `differentialRender` to the `DiffRecord (2) and the list
   of logical paragraphs (3) to produce to produce an updated list
   of rendererd paragraphs.

4. Apply `differentialIdList` to the random seet, the `DiffRecord`, and
   the initial `EditRecord` to create a new list of paragraph `ids`.

The final step is to used the information harvested in 1--4 to create
a new `EditRecord`.

```elm
Differ.update : Int -> (String -> a) -> EditRecord a -> String -> EditRecord a
Differ.update seed transformer editRecord text =
    let
        newParagraphs =
            Paragraph.logicalParagraphify text

        diffRecord =
            diff editRecord.paragraphs newParagraphs

        newRenderedParagraphs =
            differentialRender transformer diffRecord editRecord

        p =
            differentialIdList seed diffRecord editRecord
    in
    EditRecord newParagraphs newRenderedParagraphs emptyLatexState p.idList
```
