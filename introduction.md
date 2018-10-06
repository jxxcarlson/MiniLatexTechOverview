# Introduction

This document is a technical overview of the MiniLatex project.
The MiniLatex toolchain transforms LaTeX source text to HTML,
a statement that must be qualified as follows:

- It handles **subset** of LaTeX, namely, **MiniLatex**.
- The parser-renderer transforms text-mode source text:
  the macros and environments which make up MiniLatex.
  Math-mode text, e.g. $a^2 + b^2 = c^2$ is then rendered
  by MathJax, or some similar program.
- There are various renderers. One yields HTML, that is,
  a string. Another yields `Html msg`, Elm's native type for
  HTML. There is also a renderer that produces standard LaTeX.

The render-to-latex function is used to export MiniLatex
documents. It is also useful for checking
correctness. Let `f` denote the operation "parse,
then render to Latex." If `f` operates correctly,
then the idempotency
relation `f o f = f` holds. In that case,
`(f o f)(s) = f(s)` for all exemplars of
source text `s`. Thus, if equality is violated
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

## Acknowledgements

I would like to thank the many people
on the Elm Slack who have helped me with
questions on this project, and who have
made its success possible. I give special
thanks to

- Evan Czaplicki
- Ilias van Peer
- Luke Westby

Evan suggested that I look at `elm/parser` tools.
I did, and that is how this project was born.
Evan also suggested that I render to `Html a`
rather than to regular HTML This `decision` proved, like the
use of `elm/parser` to be a very good one, and it
is part of what make rendering so fast for MiniLatex.

In starting this project, I had never
used parser combinators before, so
going from the definition of the type of AST to
a functioning parser was slow-going indeed.
Ilias, whom I have never met, was unstinting
in his generous and effective help. Thankyou, Ilias!

As I made the transition from version
0.18 to 0.19 of the Elm compiler, I encountered
what looked like a blocking problem in using
MathJax to render mathematical equations.
Luke provided me with a few beautiful lines
of code that mde it all work. Wow!
