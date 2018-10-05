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
