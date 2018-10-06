# Grammar

MiniLatex has a context-sensitive grammar which will be described below.
The text below was written one year ago. Much of it may be incorrect,
so I need to review this and think about it.

## Terminals

- `spaces => sp*` where `sp`

- `ws => { sp, nl }*`, where `nl` is the newline character

- `reservedWord => { \backslash begin, \backslash end, \backslash item \}`

- `word => (Char - {sp, nl, backslash, dollarSign })+` -- Nonempty strings without whitespace characters, backslashes or dollar signs

- `specialWord => (Char - \{sp, nl, backslash, dollarSign, \& \})+`

- `macroName => (Char - {sp, nl, , leftBrace, backslash})+ - reservedWord`

Associated with these terminals are productions
`Word => word`, `Identifier => identifier`, etc.  
We shall not list all of these, but rather
just the terminal and a description of it.

## Non-Terminals

- `Arg` -- arguments for macros

- `BeginWord` -- Arg

- `BeginWord` -- produce \begin{identifier}.. \end{identifier} phrase for LaTeX environments.

- `Comment` -- LaTeX comments -- % foo, bar

- `IMath` -- inline mathematical text, as in $... $

- `DMath` -- display mathematical text, as in $$... $$

- `Env`-- LaTeX environments `\begin{foo}.. \end{foo}`

- `EnvName` -- produce an environment name such as `foo`, or `theorem`.

- `Identifier` -- a lowercase alphanumeric string beginning with a letter.

- `Item` -- item in an enumeration or itemization environemnt

- `LatexExpression` -- a custom type (algebraic data type)

- `LatexList` -- a list of LatexExpressions

- `Macro` -- a LaTeX macro of zero or more arguments.

- `MacroName` - an identifier

- `Words` - a string obtained by joining a list of Word with spaces.

## Productions

The MiniLaTeX grammar is defined by its productions. This set is fairly small, just 24 if one discounts productions which could be viewed as performing lexical analysis -- the recognition of identifiers, for instance. Of the 24 productions, 11 are for general nonterminals, 7 are for environments of various kinds, and 5 are for the tabular environment. With two exceptions we present them in EBNF form. The topmost productions are for LatexList and LatexExpression

`LatexList => LatexExpression+`

`LatexExpression => Words | Comment | IMath | DMath | Macro | Env`

The productions for the first four non-terminals on the right-hand side of the above are non-recursive, straightforward, and all result in a terminal. The terminal is of the form `Constructor x`, where the first term is a constructor for LatexExpression and `x` is a string. When `A` is a non-terminal, `A`
is the corresponding constructor.

`Words => LXString (join Word+)`

where a `Word` is any sequence of characters which is not a
whitespace character, `∖`, or `\dollar`, and where join
is the function which joins the elements of a list of strings by single spaces to produce a string.

```
Comment => Comment (Char−{\n})∗
```

```
IMath => IMath Line'
```

where Line' is Line with no occurernce of \dollar

```
DMath => DMath Line′ | DMath Line′′
```

where Line′′ is Line with no occurrence of of a right bracket.

## Macros and Environments

Let us now treat the two recursive productions. They generate LaTeX macros and environments.

`Macro => Macro Macroname Arg∗`

`Arg => Words | IMath | Macro`

Because of the use of constructions of the form

```
\begin{envName}
...
\end{envName}
```

the production of environments requires the use of a context-sensitive grammar. Here `envName`
may take on arbitrarily many values, and, in particular, values not known at the time the parser is written. Note that the production (??) below has a nonterminal followed by a terminal on the right-hand side, while (4.6) has a nonterminal followed by a terminal on the left-hand side. The presence of a terminal on the left-hand side tells us that the grammar is context-sensitive.

`EnvName => Env identifier`

`Env itemize => Environment itemize Item+`

`Env enumerate => Environment enumerate Item+`

`Env tabular => Environment tabular`

`TableEnv passThroughIdentifier => Environment passThroughIdentifier`

`TextEnv genericIdentifier => Environment genericIdentifier LatexList`

The set of all `envNames` is decomposed as follows:

`specialIdentifier =>{itemize, enumerate, tabular}`

`passThroughIdentifier => {equation, align, eqnarray, verbatim, verse}`

`genericIdentifier => identifier − specialIdentifier − passThroughIdentifier`

### Tabular Environment

The tabular environment also requires a context-sensitive grammar.

`Table =>TableHead (TableRows ncols)+`

`TableRows ncols => TableCells^ncols`

```

```
