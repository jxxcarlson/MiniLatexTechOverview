---
description: an overview of MiniLatexx
---

# Overview

The operation of MiniLatex, in rough form, is this:

```
Source text => AST => HTML
```

The AST is an _abstract syntax tree_, a kind of labeled
tree from which the source text can be derived, but which
in addition reflects an understanding of the grammar
of MiniLatex. Source text is transformed to an AST by
applying a _parser_. The second arrow is the _renderer_.
It transforms the AST to HTML, some equivalent structure.

The type of the AST is defined as follows:

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

In this overview we give some examples of how short bit of
source text are parsed and rendered, then discuss the overall
design of the renderer. In the next section we discuss the
design of the parser.

## Parsing: Examples

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

The value `LXString ("Hello, Alonzo")` is rendered to
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
a sub-renderer for component of the type of the AST. The design of the
renderer is not at all difficult.

## Rendering

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
            Html.p [ HA.style "color" "red" ] [ Html.text <| String.join "\n---\n\n" (List.map errorReport error) ]
```

## Elaborating the pipeline

As mentioned, the short pipeline `Source => AST => Html` is
rough description of the parse-render pipeline. There is in
fact quite a bit more to it.

### Chunking

The first step is to
chunk the source text into a list of "logical paragraphs."
These are either normal paragraphs or an outer begin-end
pair for an environment. Chunking is carried out by
a finite state machine that only looks at the beginnings
of lines. It is designed to be much faster than parsing,
which must look at each character.

### Applying accumulators

In the next step, an empty `LatexState` is
created. A value of this type holds information on counters
for sections, cross-references, etc. To compute these, we
use a function with the signature

```
type alias Accumulator : state -> List a -> (state, List b)
```

Thus an **Accumulator** takes a `state` and a list of `a`'s and
returns a list of `b`'s as well as a new `state`. We employ
two accumulators. The first one looks like this:

```
Accumulator.parse :
    LatexState
    -> List String
    -> ( LatexState, List (List LatexExpression) )
Accumulator.parse latexState paragraphs =
    paragraphs
        |> List.foldl parseReducer ( latexState, [] )
```

Parsing a paragraph generates a list of `LatexExpressions`,
so applying `Accumulator.parse` produces a list of lists of `LatexExpressions`, together with a `LatexState` that holds all the information needed to give sequentially
numbered sections, resolve cross-references,

A second accumulator takes the information given and applies
a renderer to each `List LatexExpression` to produce
a new pair `( LatexState, List a )`. If the renderer
has type `LatexState -> List LatexExpression -> a`, then
the list produces is a `List String`. This is the case
of rendering to standard HTML text. If the renderer
has type `LatexState -> List LatexExpression -> Html msg`, then
the list produces is a `List (Html msg)`. This is the case
of rendering to Elm's natural type representing HTML.

Here is the definition:

```
Accumulator.render :
   (LatexState -> List LatexExpression -> a)
    -> LatexState
    -> List (List LatexExpression)
    -> ( LatexState, List a )
Accumulator.render renderer latexState paragraphs =
    paragraphs
        |> List.foldl (renderReducer renderer) ( latexState, [] )
```

### Spacifying

There is subtlety to rendering a `LatexList`. The parsing
process creates a list of `LatexExpressions`. When these
are rendered the rendered results must be joined to form
good prose. The question is, should they be joined with
a space between them, or with no intervening space. The
answer is:it depends. We resolve this problem by running
the list through a `spacify` function that adds space
where needed. Then, in the end, rendered elements can
be joined end-to-end. Here is how we render a list
of `LatexExpressions`:

```
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

```
spacify : List LatexExpression -> List LatexExpression
spacify latexList =
    latexList
        |> ListMachine.runMachine addSpace
```

In this way, `renderLatexList` function prodece elements that can be
joined end-to-end.

### MathJax

The appropriately spacifyied value `Html msg`,
is sent directly to the DOM. If it contains math text,
MathJax can operate on exposed element which produce an
esthetically pleasing result.
