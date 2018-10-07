# Overview

The operation of MiniLatex, in rough form, is this:

```
Source text => AST => HTML
```

The AST is an _abstract syntax tree_, a kind of labeled
tree from which the source text can be rederived, but which
in addition reflects an understanding of the grammar
of MiniLatex. Because of this latter fact, many other
transformations are possible, e.g., transformation to
HTML.

To obtain an AST from the source text, one
applies a _parser_. The obtain HTML from
the AST, one applies another function, a
_renderer_. The type of the AST is defined as follows:

```elm
type LatexExpression
    = LXString String
    | Comment String
    | Item Int LatexExpression
    | InlineMath String
    | DisplayMath String
    | SMacro String (List LatexExpression) (List LatexExpression) LatexExpression
    | Macro String (List LatexExpression) (List LatexExpression)
    | Environment String (List LatexExpression) LatexExpression
    | LatexList (List LatexExpression)
    | LXError (List DeadEnd)
```

This type definition is the heart of MiniLatex,
and it is, in abbreviated form, the
first paragraph of code that I wrote
in developing the system.

In this overview we first give some examples of how short bits of
source text are parsed, then discuss the overall
design of the renderer. As noted above, the
`Source => AST => HTML` pipeline is a simplified
version of what is actually done. We therefore outline
the additional steps which make up the pipeline used
in production. Some of these steps have to do with
implementing features such as section numbers and
cross-references. Others have to do with performance,
that is, with speed. In the succeeding sections, e.g.,
[Parser](parser.md), [Accumulator](accumulator.md),
[Differ](differ.md), we add detail
to what is outlined below, and we discuss other issues,
such as the [Specification](specification.md) and
[Grammar](grammar.md) of MiniLatex.

## Parsing

Below are some examples of how source text is parsed and rendered.
The parsing examples were computed as in the following:

```bash
 $ elm repl
 > import Parser exposing(run)
 > import MiniLatex.Parser exposing(..)
 > run latexExpression "Hello Alonzo"
 Ok (LXString ("Hello,  Alonzo"))
    : Result (List Parser.DeadEnd) LatexExpression
```

The value `LXString ("Hello, Alonzo")` is rendered as
the string `"<span>Hello, Alonzo!</span>"`.

1.

```elm
"$a^2 + b^2 = c^2$" ==> Ok (InlineMath ("a^2 + b^2 = c^2"))
                    ==> "$a^2 + b^2 = c^2$"
```

2.

```elm
"\\italic{Wow!}" ==> Ok (Macro "italic" [] [LatexList [LXString "Wow!"]])
                 ==> "<i>Wow!</i>"
```

3.

```elm
"\\italic{" ==> Err [{ col = 9, problem = ExpectingSymbol "}", row = 1 }]
            ==> <red>ExpectingSymbol "}"</red>
```

4.

```elm
\\foo{1}{2, $x^2$} => Ok (Macro "foo" []
                          [LatexList
                             [LXString "1"]
                            , LatexList [LXString ("2, "),InlineMath "x^2"]
                          ])

                   => "<red>\\foo{1}{2, $x^2$}</red>"
```

Rendering is carried out by a function `render` which dispatches a call to
a sub-renderer for each component of the type of the AST. The design of the
renderer is not at all difficult.

## Rendering

MiniLatex has three renderers. One yields HTML, that is,
a string. Another yields `Html msg`, the native type for
HTML in Elm.
There is also a renderer that produces standard LaTeX.

The render-to-latex function is used to export MiniLatex
documents. It is also useful for checking
correctness. Let `f : String -> String` denote the operation "parse,
then render to Latex." If `f` operates correctly,
then the idempotency
relation `f o f = f` holds. In that case,
`(f o f)(s) = f(s)` for all exemplars of
source text `s`. Thus, if equality is violated
for any `s`, one knows that there is an error
in the parse-render pipeline.

In this document we discusson only the renderer with
`Html msg` as target. Here is the top-level rendering
function:

```elm
render : LatexState -> LatexExpression -> Html msg
render latexState latexExpression =
    case latexExpression of
        Comment str ->
            Html.p [] [ Html.text <| "" ]

        Macro name optArgs args ->
            renderMacro latexState name optArgs args

        SMacro name optArgs args le ->
            renderSMacro latexState name optArgs args le

        Item level latexExpr ->
            renderItem latexState level latexExpr

        InlineMath str ->
            Html.span [] [ oneSpace, inlineMathText str ]

        DisplayMath str ->
            displayMathText str

        Environment name args body ->
            renderEnvironment latexState name args body

        LatexList latexList ->
            renderLatexList latexState latexList

        LXString str ->
            case String.left 1 str of
                " " ->
                    Html.span [ HA.style "margin-left" "1px" ] [ Html.text str ]

                _ ->
                    Html.span [] [ Html.text str ]

        LXError error ->
            Html.p
              [ HA.style "color" "red" ]
              [ Html.text <| String.join "\n---\n\n" (List.map errorReport error) ]
```

## Elaborating the pipeline

As mentioned, the short pipeline `Source => AST => Html` is a
rough description of the parse-render pipeline. There is in
fact quite a bit more to it. In outline, here is the process:

```elm
Source text
   => List of paragraphs                           -- (1) Paragraph.paragraphify
   => (LatexState, List (List LatexExpression))    -- (2) Accumulator.parse
   => ( LatexState, List (Html msg))               -- (3) Accumulator.render
   =>  Html msg                                    -- (4) Concatenate
   => DOM                                          -- (5) Elm runtime
   => DOM                                          -- (6) MathJax
```

### (1) Paragraphify

The function `Paragraph.paragraphify` "chunks"
source text into a list of logical paragraphs:
`String -> List String`. Logical paragraphs are either
normal paragraphs or an outer begin-end
pair of an environment. Chunking is carried out by
a finite state machine that only looks at the beginnings
of lines. It is designed to be much faster than parsing,
which must look at each character.

### (2) `Accumulate.parse`

This step is an elarboration of parsing. Parsing
a string produces a `List LatexExpression`,
and mapping the parser onto a list of strings
produces a `List (List LatexExpression)`.
The function `Accumulator.parse` takes as
arguments a `LatexState` and a `List String`,
producing a pair consisiting of an updated
`LatexState` and the `List (List LatexExpression)`
just described. A value of type `LatexState` holds
counters for section and subsection numbers,
information for cross-references, etc. If one
applies `Accumulator.parse` to an empty `LatexState`
and a list of strings, the final `LatexState` carries
the information needed to render sections with
sequential numbers, resolve cross-references, etc.

```elm
Accumlator.parse :
    LatexState
    -> List String
    -> ( LatexState, List (List LatexExpression) )
```

The general pattern is

```elm
type alias Accumulator : state -> List a -> (state, List b)
```

An accumulator takes a `state` and a list of `a`'s and
returns an updated `state` and a list of `b`'s.

### (3) Accumulator.render

Consider a function of the type:

```elm
Accumulator.render :
   (LatexState -> List LatexExpression -> a)
    -> LatexState
    -> List (List LatexExpression)
    -> ( LatexState, List a )
```

The first argument, takes`LatexSate` and a list of
`LatexExpressions` as arguments and returns a value
of type `a`. It is a rendering function that renders
to type `a`. Given such a rendering function, we obtain
an accumulator which transforms a `List (List LatexExpression)`
to a `List a`:

```elm
LatexState -> List (List LatexExpression) -> ( LatexState, List a )
```

In the outline above, `a = Html msg`.

### (4) Concatenate

This is a simple step. A value `list` of type `List (Html msg`) must
be converted to a type of `Html msg` by converting each element
of a list into the `Html msg` representation of an HTML paragraph,
and then converting this list of paragraphs into the `Html msg`
represantion of a single HTML div.

### (5) Elm Runtime

The Elm Runtime converts the `Html msg` value into node
in the DOM of the brower.

### (6) MathJax

MathJax renders the DOM nodes that carry mathematical text.

## Spacifying

There is subtlety to rendering a `LatexList`. The parsing
process creates a list of `LatexExpressions`. When these
are rendered, the rendered results must be joined to form
good prose. The question is, should they be joined with
a space between them, or with no intervening space. The
answer is: it depends. We resolve this problem by running
the list through a `spacify` function that adds space
where needed. Then, in the end, rendered elements can
be joined end-to-end. Here is how we render a list
of `LatexExpressions`:

```elm
renderLatexList : LatexState -> List LatexExpression -> Html msg
renderLatexList latexState latexList =
    latexList
        |> spacify
        |> List.map (render latexState)
        |> (\list -> Html.span [ HA.style "margin-bottom" "10px" ] list)
```

Spacifying is carried out by
a "List machine," as described in
[this post on Medium](https://medium.com/me/stats/post/c07700bba13c).
A List Machine reads a tape, just as does a Turing machine,
but it only has access to the current square on the tape,
the one before it, and the one after it.

```elm
spacify : List LatexExpression -> List LatexExpression
spacify latexList =
    latexList
        |> ListMachine.runMachine addSpace
```

In this way, `renderLatexList` function produces elements that can be
joined end-to-end.

## Difffing

There is one additional elaboration of the above. The typical workflow
of a user editing a MiniLatex document is to open it up, then begin
adding/changing bits of text here and there. In a document management system
like [knode.io](https://knode.io) which hosts MiniLatex, the document
is automatically rendered every 250 milliseconds. For long documents,
there could be a noticeable delay, which of course is quite annoying.
To prevent this, we employ a very simple diffing strategy. It is far
from optimal in theory, but it works quite well in practice because
of the way human editors typically operate: by
mostly making small, localized changes.

Implementing this strategy has an upside and a downside. The upside
is that changes are almost always rendered within the 250 millisecond
window, providing a very good user experience -- "live editing and rendering."
The downside is that without re-rendering the entire document, one cannot
properly resolve cross-references, compute section numbers, etc.
However, the user can command a full render whenever it is needed (control-F
in [knode.io](https://knode.io)).

The strategy is this. Let `u` and `v` be two lists
of strings. Write them as `u = a x b`, `v = u y b`,
where `a` is the greatest common prefix and `b` is the
greatest common suffix. Supposed that we have in hand
`u' = render u = a' x' b'`, where `a' = render a`, etc.
Let `y' = render y`. Then `v' = diff u = a' y' b'`.

To carry out this strategy, one introduces a new type alias,
`DiffRecord`, which holds `a`, `x`, `b`, and `y`, and one
defines a function

```
diff : List String -> List String -> DiffRecord
```

One also defines the notion of an `EditRecord` to
compute `v = a' y' b'` in an efficient way from
`u = a' x' b'` and the `DiffRecord`, that
is, by computing only `y => y'`. See the [Differ](differ.md)
sections for more details on both transformations.
