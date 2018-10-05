# Differ

MiniLatex uses a number of optimizations. One of these
is to re-parser only paragraphs which have. This is
relevant to the editing process, since instantaneous
feedback is essential for a satisfying user experience.

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

## Implementing the diff stategy

The stuctures `u = a x b`, `v = u y b` map directly
onto the `DiffRecord` type:

```
type alias DiffRecord =
    { commonInitialSegment : List String
    , commonTerminalSegment : List String
    , middleSegmentInSource : List String
    , middleSegmentInTarget : List String
    }
```

The implementation of

```
diff : List String -> List String -> DiffRecord
```

is straightforward.

## Edit records

The function `diff` carries out the transformation `a x b => a y b`.
To effect `a' x' b' => a' y' b'`, one needs the notion of an
`EditRecord`, which carries the needed state:

```
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

Below is the definition of `differentialRender`. It takes
a rendering function, a `DiffRecord`, and an `EditRecord`
as input and produces a list of rendered paragraphs as output.

```
differentialRender : (String -> a) -> DiffRecord -> EditRecord a -> List a
differentialRender renderer diffRecord editRecord =
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

```
update : Int -> (String -> a) -> EditRecord a -> String -> EditRecord a
update seed transformer editRecord text =
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
    EditRecord newParagraphs newRenderedParagraphs emptyLatexState p.idList p.newIdsStart p.newIdsEnd
```
