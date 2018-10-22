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

### LaTeX

Macros in this group of 24 exist in LaTeX
and have the same meaning.

```
author
date
email
cite
emph
eqref
href
index
label
maketitle
revision
section
section*
medskip
ref
smallskip
subsection
subsection*
subsubsection
subsubsection*
subheading
tableofcontents
term
title
```

### MiniLaTeX I

Macros in this group of 19 are not in LaTeX,
but their definitions are in the
standard MiniLatex input file.

```
dollar
percent
code
image
imageref
italic
mdash
ndash
underscore
backslash
texarg
setcounter
innertableofcontents
red
blue
highlight
strike
strong
xlink
```

### MiniLatex II

Macros in this group of 2 have no counterpart
in basic LaTeX or in one of its packages.

```
ellie
iframe
```

## Supproted Environments

### LaTeX environments

The 12 environments below are supported in MiniLatex

```
center
comment
enumerate
eqnarray
equation
indent
itemize
quotation
tabular
thebibliography
verbatim
verse
```

### MiniLatex Environments

```
listing
```
