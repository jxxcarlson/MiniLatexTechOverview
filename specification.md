# Specification

This section is incomplete and is very much a work in progress. Below is a sketch.

## What is supported

1. Familiar macros like `\section`, `\subsection`,
   `\emph`, `\cite`, etc. See the list below.

2. Familiar environments such as `theorem`,
   `equation`, `verbatim`, `enumerate`, `itemize`,
   `tabular`, etc. See the list below.

3. Math mode macros

## Special macros and environments

There are some macros and environments which
are unique to MiniLatex. When a MiniLatex
document is exported, it is exported along
with a file of text-mode macro definitions so that
the document can be rendered using
`pdflatex`.

## What is not supported

1. The elements `--` and `---` are not supported. Use
   `\ndash` and `\mdash` instead.

2. Regions, such as `{\bf la di dah!}. In this case, use`\strong{La di dah}`

3. Text mode macros

4. Stylesheets

## Supported Macros

TODO

## Supproted Environments

TODO
