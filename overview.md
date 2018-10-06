---
description: an overview of MiniLatexx
---

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
HTML.Source text is transformed to an AST by
applying a _parser_. The second arrow, from AST to HTML,
is the _renderer_.

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

In this overview we first give some examples of how short bits of
source text are parsed, then discuss the overall
design of the renderer. As noted above, the
`Source => AST => HTML` pipeline is a simplified
version of what is actually done. We therefore outline
the additional steps which make up the pipeline used
in production. Some of these steps have to do with
implementing features such as section numbers and
cross-references. Others have to do with perfomrance,
that is, with speed. The succeeding sections add detail
to what is outlined below.

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
a sub-renderer for each component of the type of the AST. The design of the
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

## <a name="elaborating"> Elaborating the pipeline

As mentioned, the short pipeline `Source => AST => Html` is a
rough description of the parse-render pipeline. There is in
fact quite a bit more to it. In outline, here is the process:

```elm
Source text
   => List of paragraphs                           -- Paragraph.paragraphify
   => (LatexState, List (List LatexExpression))    -- Accumulator.parse
   => ( LatexState, List (Html msg))               -- Accumulator.render
   =>  Html msg                                    -- |> Html.div []
   => DOM                                          -- Elm runtime
   => DOM                                          -- MathJax
```

### Chunking

Chunk source text into a list of logical paragraphs:
`String -> List String`. Logical paragraphs are either
normal paragraphs or an outer begin-end
pair of an environment. Chunking is carried out by
a finite state machine that only looks at the beginnings
of lines. It is designed to be much faster than parsing,
which must look at each character.

### `Accumulate.parse`

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

### Accumulator.render

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

### Spacifying

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

### MathJax

The appropriately spacifyied value `Html msg`,
is sent directly to the DOM. If it contains math text,
MathJax can operate on exposed element which produce an
esthetically pleasing result.
