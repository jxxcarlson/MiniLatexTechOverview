# Parser

The MiniLatex parser is written in Elm using
Evan Czaplicki's [parser combinator package](https://package.elm-lang.org/packages/elm/parser/latest/).
We will take a look at some of the top level parsers, then examine in detail
the parser for environments, which is both the most important and the most complex one.

## Top level parsers

Here is the top-level parsing function:

```elm
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

The `oneOf` parser tries each parser in its argument list in turn, succeeding by
applying the first parser to succeed, and failing otherwise.

```
oneOf : List (Parser a) -> Parser a
```

The expression `lazy (\_ -> environment)` is needed to ensure
that recursion works. If one reads down the argument list of `oneOf`, one can extract a production
for a grammar for MiniLatex:

```elm
LatexExpression -> Comment
                 | DisplayMath
                 | InlineMath
                 | Macro
                 | SMacro
                 | LXString
                 | Environment env
```

We will come back to this later when we discuss the MiniLatex grammar.
There is no one-to-one correspondence between parser functions and productions,
but there is a close relations.

### Macro parser

Let us look at one of the components of the `latexExpression` parser, say,
the parsers for macros -- expressions like
like `\italic{Wow!}`. For this we use the`elm/parser` pipeline construction,
which sequences parsers:

```elm
macro : Parser () -> Parser LatexExpression
macro wsParser =
    succeed Macro
        |= macroName
        |= itemList optionalArg
        |= itemList arg
        |. wsParser
```

The rough idea of the pipeline is to apply the parsers
`macroName`, `itemList optionalArg`, `itemList arg`,
and `wsParser` in sequence, chomping away at the input,
keeping the parse result in the case of `|=` and ignoring
it in the case `|.`

```elm
(|=) : Parser (a -> b) -> Parser a -> Parser b
(|.) : Parser keep -> Parser ignore -> Parser keep
```

In the case at hand, the pipeline
will feed three arguments to the type constructor
`Macro`. Since `succeed` has type `succeed : a -> Parser a`,
the result will be a `Parser LatexExpression`.

Note that the `macro` function takes a parser as input
and produces a `Parser LatexExpression` as output.
The input, which has type `Parser ()` should parse
white space -- perhaps just `' '`, or perhaps
newlines as well. This definition of `macro`
gives added flexibility.

As with the top level parser, we can derive a production for the
MiniLatex grammar directlh from the defintion of the parser:

```elm
Macro -> MacroName | OptionalArg* | Arg*
```

Rather than pursuing the rabbit down its hole to explain
the parsers `macroName`, `itemList`, `optionalArg`, `arg`,
and the possible whitespace parsers, we refer to
the [source code](https://github.com/jxxcarlson/meenylatex/blob/master/src/MiniLatex/Parser.elm).

## The environment parser

The environnment parser is the most complex of the parsers.
We give a simplified version, then discuss the changes
needed to obtain the actual parser.

```elm
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

```elm
andThen : (a -> Parser b) -> Parser a -> Parser b
```

Consider now the second parser which makes up `environment`

```elm
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

Below is the definition of `environmentParser`. It is simple
application of the parser pipeline construct.

```elm
environmentParser : String -> String -> Parser LatexExpression
environmentParser endWord_ envType =
  succeed (Environment envType)
    |. ws
    |= (nonemptyItemList latexExpression) |> map LatexList)
    |. ws
    |. symbol endWord_
    |. ws
```

## Digression: Monads in Elm

Although Elm does not explicitly mention the much (and mistakenly) feared
notion of monad, monads are implicit in Elm. Indeed, the `andThen` function
is a variant of the bind function for the `Parser` monad.
Compare the type signatures for `andThen` and the monadic bind operator
`>>=`:

```
andThen : (a -> Parser b) -> Parser a -> Parser b
(>>=) : M a -> (a -> M b) -> M b
```

Up to a permutation and renaming of arguments, they are the same.

### Note

My favorite way of thinking about **bind** is to use the function

```elm
beta : (a -> M b) -> M a -> M b
```

Consider first functions `f : a -> b` and `g : b -> c`.
They can be composed: we can form `g o f : a -> c`.
Now let `f : a -> M b` and `g : b -> M c` be "monadic" functions,
where `M` is a type constructor (Elm) or a functor (Haskell).
They cannot be composed as is. However, with the aid of
`beta`, they can be:

```elm
g o' f = \x -> beta g (f x)
```

**Conclusion:** beta gives a way of composing monadic functions `f`
and `g`, where `M` is the monad. Bind does the same thing:

```
g o' f = \x -> (f x) >>= g
```

All the rigamarole about functors, monads, etc., is important,
because the monad conditions on `M` guarantee that the new composition
operator `o'` does what we expect it to, e.g., obey the associative law:

```
h o' (g o' f) = (h o' g) o' f
```

This is the point of view of Kliesli categories, which I learned
from the writings and videos of Bartosz Miliewski:
