# Introduction

This document is a technical overview of the MiniLatex project.
The MiniLatex toolchain transforms LaTeX source text to HTML,
a statement that must be qualified as follows:

- It handles **subset** of LaTeX, namely, **MiniLatex**.
- The parser-renderer transforms text-mode source text:
  the macros and environments which make up MiniLatex.
  Math-mode test, e.g. $a^2 + b^2 = c^2$ is then rendered
  by MathJax, or some similar program.
- There are various renderers. One which yields HTML, that is,
  a string. Another yields `Html msg`, Elm's native type for
  HTML. There is also a renders the produces standard LaTeX.

The render-to-latex function is used to export MiniLatex
documents. It is also useful for checking
correctness. Let `f` denote the operation "parse,
then render to Latex." Then the idempotency
relation `f o f = f` holds. Thus, if `s` is
a piece of source text, then one should have
`(f o f)(s) = f(s)`. If equality is violated
for any `s`, one knows that there is an error
in the parse-render pipeline.

For a demonstration of this technology, see
[MiniLatex Live](https://jxxcarlson.github.io/app/miniLatexLive/index.html).
For a full content-management system based on MiniLatex, see
[knode.io](https://knode.io)

MiniLatex is written in [Elm](http://elm-lang.org/),
a statically typed functional language. The code
is available both on [GitHub](https://github.com/jxxcarlson/meenylatex)
and at [package.elm-lang.org](https://package.elm-lang.org/packages/jxxcarlson/meenylatex/latest/)

## Origin and Acknowledgements

The MiniLatex project would never have succeeded without
the generous help and advice of many people. At the Elm
Europe 2017, I had a short conversation with Evan Czaplicki
about parsing. At that time I was still working on an
extension to Asciidoc which gives it LaTeX-like features,
and I was thinking of writing a new Asciidoc parser.
A realized not long after that I could do something
that I had not dared dream of: write a parser for
a subset of LaTeX. I wrote down a preliminary definition
of the type of the abstract syntax tree (AST) and began
working on the parser. This was the first time that
I had worked with parser combinators. Fortunately, I was
able to get many of my questions answered on the Elm Slack.
I would particularly like to acknowledge Ilias van Peer, whose
generous help was crucial.

Another phase was the transition from version 0.18 to 0.19
of the Elm compiler. This was important because performance
optimizations in the compiler made possible another dream:
real-time parsing and rendering of a subset of LaTeX.
The transition required some substantial changes, and I was
blocked on how to implement rendering. Evan
Czaplicki suggested that I replace rendering
to a string of HTML text by rendering to `Html msg`,
Elm's native data type that represents HTML.
This not only solved the problem, but also
yielded a significant performance boost.

The final rendering step (MathJax), has always been
delicate/tricky, and it also required a substantial
change. I am indebted to Luke Westby for
an elegant solution to this problem.
