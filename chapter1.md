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
