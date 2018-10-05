# Parser

The MiniLatex parser is written in Elm using
Evan Czaplicki's [parser combinator package](https://package.elm-lang.org/packages/elm/parser/latest/).
Here is the top-level parsing function:

```
latexExpression : Parser LatexExpression
latexExpression =
    oneOf
        [ texComment
        , displayMathDollar
        , displayMathBrackets
        , inlineMath ws
        , macro ws
        , smacro
        , words
        , lazy (\_ -> environment)
        ]
```

If one reads down the argument list of `oneOf`, one can extract this production
for a grammar for MiniLatex:

```
LatexExpression -> Comment
                 | DisplayMath
                 | InlineMath
                 | Macro
                 | SMacro
                 | LXString
                 | Environment env
```

We will come back to this later when we discuss the MiniLatex grammar.
Let us look at some other parsing functions.

```
macro : Parser () -> Parser LatexExpression
macro wsParser =
    succeed Macro
        |= macroName
        |= itemList optionalArg
        |= itemList arg
        |. wsParser
```

The `macro` function takes a parser as input
and produces a `Parser LatexExpression` as output.
The input, which has type `Parser ()` should parse
white space -- perhaps just `' '`, or perhaps
newlines as well. The `succeed` funtion has
signature

```
succeed : a -> Parser a
```

`Macro` in this context is a type constructor,
so it will return a `Macro` value given the
proper arguments. This is what the "parser pipeline"
does. A parser pipeline is built out of the symbols
`|=` and `|.`, where the first means "parse and keep the result,"
and where the second means "parse and ignore the result."
In this case, the parser pipeline will "eat" the name of the macro,
the list of optional arguments, and the list of regular arguments,
passing these as arguments to `Macro`. The pipeline ignores
any trailing whitespace.
