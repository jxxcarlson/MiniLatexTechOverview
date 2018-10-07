# Introduction

This document is a technical overview of the MiniLatex project.
The project consists of three elements:

1. A [specification](specification.md) and [grammar](grammar.md)
   for a subset of LaTeX called MiniLatex

2. A parser-renderer package that can be used
   to parse MiniLaTeX and render it to HTML. The code
   is open-source: see
   [Github](https://github.com/jxxcarlson/meenylatex)
   or [package.elm-lang.org](https://package.elm-lang.org/packages/jxxcarlson/meenylatex/latest/)

3. A content management system which hosts MiniLatex and provides
   user authentication, a searchable store of MiniLatex documents, etc.
   See [knode.io](https://knode.io). Documents written using `knode.io`
   can be made public, in which case users can search for them
   and read them withought signing in. See, for example,
   [knode.io/427](https://knode.io/427)

For a demonstration of this technology, see
[MiniLatex Live](https://jxxcarlson.github.io/app/miniLatexLive/index.html).

MiniLatex is written in [Elm](http://elm-lang.org/),
a statically typed functional language created by Evan Czaplicki.
Elm is a language whose target use is the development of web apps.
Its type system gives unparalled assurance that code, once deployed,
"just works," as well as the ability to refactor without fear and
therefore to write and maintain code for the long term. These
facts, combined with the expressivity of Elm's type system and the quality
of its parser combinator library ([elm/parser](https://package.elm-lang.org/packages/elm/parser/latest/)), makes it an ideal vehicle
for the MiniLatex project.

**Plans.** MiniLatex is still a research project and knode.io is still
in development. That said,
I've used MiniLatex on knode.io to write lecture notes, e.g, [knode.io/427](https://knode.io/427).
So it is ready to be used by brave souls. If you would like to experiment
with it, by all means do so. I very much value your feedback.
You can write to me at jxxcarlson@gmail.com.
See the [Plans](plans.md) section for more information about where
MiniLatex is going.

## Acknowledgements

I would like to thank the many people
on the Elm Slack who have helped me with
questions on this project, and who have
made its success possible. I give special
thanks to

- Evan Czaplicki
- Ilias van Peer
- Luke Westby

Evan suggested that I look at `elm/parser` tools
for building a parser for another project.
I did, and after a few weeks of study, I realized
that I could attempt something much more ambitious:
a parser for subset of LaTeX. That is how this project
was born. Evan later suggested that I render to `Html a`
rather than to regular HTML This change is part of
what make rendering so very fast.

In starting this project, I had never
used parser combinators before, so
going from the definition of the type of the AST to
a functioning parser was at first quite a struggle.
The generous help that I received rom Ilias, whom I have never met,
other than through the Elm Slack, helped me to
find my way in this unfamiliar terrain.

As I made the transition from version
0.18 to 0.19 of the Elm compiler, I encountered
what looked like a blocking problem in using
MathJax to render mathematical equations.
Luke provided me with a few beautiful lines
of code that mde it all work.
