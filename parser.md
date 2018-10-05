# Parser

The MiniLatex parser is written in Elm using
Evan Czaplicki's [parser combinator package](https://package.elm-lang.org/packages/elm/parser/latest/).
We will take a look at some of the top level parsers, then examine in detail
the parser for environments, which is both the most important and the most complex one.

## Top level parsers

Here is the top-level parsing function:

```haskell
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

```haskell
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

```haskell
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

```haskell
succeed : a -> Parser a
```

`Macro` in this context is a type constructor,
so it will return a `Macro` value given the
proper arguments.The "parser pipeline,"
built out of the symbols
`|=` and `|.` does just this: it sequences
the action of a list of parsers, `macroName`,
`itemList optionalArg`, etc. The symbol
`|=` means "apply the parser on the right and keep the result."
The symbol `|.` means "apply the parser on the right and ignore the result."
In this case, the parser pipeline will "eat" the name of the macro,
the list of optional arguments, and the list of regular arguments,
passing these to `Macro`. The pipeline ignores
any trailing whitespace. Here are the signatures of the parser pipeline
operators:

```haskell
(|=) : Parser (a -> b) -> Parser a -> Parser b
(|.) : Parser keep -> Parser ignore -> Parser keep
```

As with the top level parser, we can derive a production for the
MiniLatex grammar:

```haskell
Macro -> MacroName | OptionalArg* | Arg*
```

## The environment parser

The environnment parser is the most complex of the parsers.
We give a simplified version, then discuss the changes
needed to obtain the actual parser.

```haskell
environment : Parser LatexExpression
environment =
  envName |> andThen environmentOfType
```

The envName parser recognizes text of the form `\\begin{foo}`, extracting
the string `foo`:

```el
envName : Parser String
envName =
  succeed identity
    |. PH.spaces
    |. symbol "\\begin{"
    |= parseToSymbol "}"
```

There is a companion parser `endWord`, which recognizes text like
`\\end{foo}`. The parser `andThen` is used to sequence parsers:

```haskell
andThen : (a -> Parser b) -> Parser a -> Parser b
```

Consider now the second parser which makes up `environment`

```haskell
environmentOfType : String -> Parser LatexExpression
environmentOfType envType =
  let
    theEndWord = "\\end{" ++ envType ++ "}"
  in
    environmentParse theEndWord envType
```

One sees that the types match, since

```
envName |> andThen environmentOfType == andThen environmentOfType envName
```

The result is that the information gathered by `environment` is passed
to `environmentParser` with arguments of the form `\\end{foo}` and `foo`.
At this point the use of the two arguments is redundant. However,
for the "real" parser, `envType` needs to be transformed: in some cases,
it is passed through as is, while in others it is changed.

```haskell
environmentParser : String -> String -> Parser LatexExpression
environmentParser endWord_ envType =
  succeed (Environment envType)
    |. ws
    |= (nonemptyItemList latexExpression) |> map LatexList)
    |. ws
    |. symbol endWord_
    |. ws
```

## Monads in Elm

The `andThen` function is a variant of the bind function for the `Parser` monad.
Compare the type signatures:

```
andThen : (a -> Parser b) -> Parser a -> Parser b
(>>=) : M b -> (b -> Mb) -> M c
```

My favorite way of thinking about **bind** is to use the function

```haskell
beta : (b -> Mc) -> M b -> M c
```

Consider functions `f : a -> M b` and `g : b -> M c`. Then
one can define

```haskell
g o f = \x -> beta g (f x)
```

Thus bind/beta gives a way of composing "monadic" functions `f`
and `g`, where `M` is the monad.
