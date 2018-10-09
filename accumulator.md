# Accumulators

As noted in the overview, one uses an accumulator to
collect information about sequentially numbered
sections, cross-references, tables of content, etc.,
then uses a second accumulator to render the source text with these features.
Accumulators are made of up of reducers and folds:

```elm
type alias Reducer : a -> b -> b
List.foldl : (a -> b -> b ) -> b -> List a -> b
```

A `Reducer` is a name for the type of the
first argument of a fold. Consider a reducer of the
form

```elm
Reducer a b = a -> (state, List b) -> (state, List b)
```

It fits into a fold of the form

```elm
StateReducer a b -> (state, List b) -> List a -> (state, List b)
```

Let `transform` have type `StateReducer a b`. Define

```elm
acc transformer state_ inputList =
  List.foldl transformer (state_, []) inputList
```

The type of this function is

```elm
Accumulator a b = State -> List a -> (State, List b)
```

To restate in plainer English, an `Accumulator a b` takes
as input a `State a b` and a `List a` and returns a tuple
consisting of an updated `State a b` and another list, one
of type `List b`.

## Accumulator.parse

Let us discuss the accumulators used in MiniLatex.
The first of these is `Accumulator.parse`. Its function
is to take a `LatexState` and a list of paragraphs, i.e.,
a `List String`, and produce an updated `LatexState`
and a `List (List LatexExpression)`. Each element
of the latter is a `List LatexExpression` representing
the application of the MiniLatex parser to a paragraph.

```
Accumulator.parse :
    LatexState
    -> List String
    -> ( LatexState, List (List LatexExpression) )
Accumulator.parse latexState paragraphs =
    paragraphs
        |> List.foldl parseReducer ( latexState, [] )
```

The `Accumulator.parse` function applies a `List.foldl`
to a pair consisting of an initial `LatexState` and
an empty list using a `parseReducer`. The latter is
defined as follows:

```
parseReducer :
    String
    -> ( LatexState, List (List LatexExpression) )
    -> ( LatexState, List (List LatexExpression) )
parseReducer inputString ( latexState, inputList ) =
    let
        parsedInput =
            Parser.parse inputString

        newLatexState =
            latexStateReducer parsedInput latexState
    in
    ( newLatexState, inputList ++ [ parsedInput ] )
```

The `parseReducer` takes as input a string, representing a
logical paragraph of source text, and a pair consisting of
a `LatexState` and a `List (List LatexExpression`. It
parses the string and computes a new `LatexState` using
`latexStateReducer`:

```
latexStateReducer : List LatexExpression -> LatexState -> LatexState
```

We discuss this function later on.
The return value of the `parseReducer` is
a pair consisting of the new `LatexState` and
inputList with the parsedInput appended.

## Accumulator.render

`Accumulator.render renderer` is an accumulator
which takes a `LatexState` and a `List (List LatexExpression)`
as input, and produces a new `LatexState` and `List a`
as output. The nature of `a` depends on the `renderer`
function used:

```
Accumulator.render :
    (LatexState -> List LatexExpression -> a)
    -> LatexState
    -> List (List LatexExpression)
    -> ( LatexState, List a )
Accumulator.render renderer latexState paragraphs =
    paragraphs
        |> List.foldl (renderReducer renderer) ( latexState, [] )
```

The expression `renderReducer renderer` is a reducer of type

```
 List LatexExpression  -> ( LatexState, List a ) -> ( LatexState, List a )
```

It operates as follows:

1. Compute a new `LatexState` by applying `latexStateReducer`
   to (a) the given list of `LatexExpressions` and (b)
   the `LatexState` coming from the first element of the
   second argument, a tuple of type `(LatexState, List a)`

2. Apply the `renderer` to the new state computed in the previous
   step and the `List LatexExpression` given by the first argument.

3. Return a tuple consisting of the updated `LatexState` and
   the `inputList` with the newly rendered text appended.

```
renderReducer :
    (LatexState -> List LatexExpression -> a)
    -> List LatexExpression
    -> ( LatexState, List a )
    -> ( LatexState, List a )
renderReducer renderer listLatexExpression ( latexState, inputList ) =
    let
        newLatexState =
            latexStateReducer listLatexExpression latexState

        renderedInput =
            renderer newState listLatexExpression
    in
    ( newLatexState, inputList ++ [ renderedInput ] )
```

## LatexStateReducer

The purpose of the `latexStateReducer` is to update a given
`LatexState` using the information contained in a `List LatexExpression`,
i.e., the parse result of a paragraph.

```
latexStateReducer : List LatexExpression -> LatexState -> LatexState
latexStateReducer parsedParagraph latexState =
    let
        theInfo =
            parsedParagraph
                |> List.head
                |> Maybe.map info
                |> Maybe.withDefault (LatexInfo "null" "null" [] [])
    in
    (latexStateReducerDispatcher  theInfo) theInfo latexState
```

This a fairly complex reducer. In brief, it looks at the first
`LatexExpression` in the `List LatexExpression` representing a
given paragraph, then uses the function `info` to compute a record, `theInfo`,
which extracts certain information from the paragraph. For example,
if the paragraph is `\section{Introduction}`, the `theInfo` will
contatin fields `name = section` and `typ = macro`. The call
`latexStateReducerDispatcher theInfo` uses a dictionary lookup to
produce a function of type `LatexInfo -> LatexState -> LatexState` from `theInfo`. Therefore
the value of the expression

```
(latexStateReducerDispatcher theInfo) theInfo latexStae
```

is of type `LatexState`, in accord with what one reads from the type annotation of
`latexStateReducer parsedParagraph latexState`.

Let's see how this works in a simple example:

```
> import MiniLatex.Parser exposing(..)
> import Parser exposing(run)
> import MiniLatex.Accumulator exposing(info)
> result = run latexExpression "\\section{foo}"
Ok (Macro "section" [] [LatexList [LXString "foo"]])
> expression = Macro "section" [] [LatexList [LXString "foo"]]
> theInfo = info expression
    { name = "section", options = [], typ = "macro", value = [LatexList [LXString "Foo"]] }
    : MiniLatex.Accumulator.LatexInfo
```

Now the dispatcher looks like this

```
latexStateReducerDispatcher : LatexInfo -> (LatexInfo -> LatexState -> LatexState)
latexStateReducerDispatcher theInfo =
    case Dict.get ( theInfo.typ, theInfo.name ) latexStateReducerDict of
        Just f ->
            f

        Nothing ->
            \latexInfo latexState -> latexState
```

where the dictionary has type

```
latexStateReducerDict : Dict.Dict ( String, String ) (LatexInfo -> LatexState -> LatexState)
```

If the key given by `theInfo` is not in the `latexStateReducerDict`,
projection of the arguments to `LatexState` is returned. In the case at hand,
`theInfo` has fields `name = "section"` and `typ = "macro"`,
and the function returned by the dictionary is `SRH.updateSectionNumber x y`.
Here `SRH` is an alias of the `StateReducerHelper` module. Referring to that module, we find that

```
updateSectionNumber : LatexInfo -> LatexState -> LatexState
updateSectionNumber info latexState =
    let
        label =
            getCounter "s1" latexState |> (\x -> x + 1) |> String.fromInt
    in
        latexState
            |> incrementCounter "s1"
            |> updateCounter "s2" 0
            |> updateCounter "s3" 0
            |> addSection (PT.unpackString info.value) label 1
```

Thus `updateSectionNumber` increments the "s1" counter in the `latexState`
and sets the other section counters to zero. The `addSection` function,
listed below, adds the current section to the table of contents field
of the `LatexState`:

```
LatexState.addSection : String -> String -> Int -> LatexState -> LatexState
LatexState.addSection sectionName label level latexState =
    let
        newEntry =
            TocEntry sectionName label level

        toc =
            latexState.tableOfContents ++ [ newEntry ]
    in
    { latexState | tableOfContents = toc }
```

We will not go down the rabbit hole any further, but you get the general idea.
The `LatexState` holds counters, the table of contents, cross references,
and a general-purpose dictionaray. To add new abilities to `MiniLatex`,
one can add new entries to `latexStateReducerDict` or new fields to
`LatexState`.

```
type alias LatexState =
    { counters : Counters
    , crossReferences : CrossReferences
    , tableOfContents : TableOfContents
    , dictionary : Dictionary
    }
```

### Example: rendering a section

To conclude this discussion, we exhibit the render for sections:
s

```
renderSection : LatexState -> List LatexExpression -> Html msg
renderSection latexState args =
    let
        sectionName =
            MiniLatex.Render.renderArg 0 latexState args

        s1 =
            getCounter "s1" latexState

        label =
            if s1 > 0 then
                String.fromInt s1 ++ " "

            else
                ""

        ref =
            idPhrase "section" sectionName
    in
    Html.h2 (headingStyle ref 24) [ Html.text <| label ++ sectionName ]
```

### Appendix: the latexStateReducerDict

```
latexStateReducerDict : Dict.Dict ( String, String ) (LatexInfo -> LatexState -> LatexState)
latexStateReducerDict =
    Dict.fromList
        [ ( ( "macro", "setcounter" ), \x y -> SRH.setSectionCounters x y )
        , ( ( "macro", "section" ), \x y -> SRH.updateSectionNumber x y )
        , ( ( "macro", "subsection" ), \x y -> SRH.updateSubsectionNumber x y )
        , ( ( "macro", "subsubsection" ), \x y -> SRH.updateSubsubsectionNumber x y )
        , ( ( "macro", "title" ), \x y -> SRH.setDictionaryItemForMacro x y )
        , ( ( "macro", "author" ), \x y -> SRH.setDictionaryItemForMacro x y )
        , ( ( "macro", "date" ), \x y -> SRH.setDictionaryItemForMacro x y )
        , ( ( "macro", "email" ), \x y -> SRH.setDictionaryItemForMacro x y )
        , ( ( "macro", "host" ), \x y -> SRH.setDictionaryItemForMacro x y )
        , ( ( "macro", "setclient" ), \x y -> SRH.setDictionaryItemForMacro x y )
        , ( ( "macro", "setdocid" ), \x y -> SRH.setDictionaryItemForMacro x y )
        , ( ( "macro", "revision" ), \x y -> SRH.setDictionaryItemForMacro x y )
        , ( ( "env", "theorem" ), \x y -> SRH.setTheoremNumber x y )
        , ( ( "env", "proposition" ), \x y -> SRH.setTheoremNumber x y )
        , ( ( "env", "lemma" ), \x y -> SRH.setTheoremNumber x y )
        , ( ( "env", "definition" ), \x y -> SRH.setTheoremNumber x y )
        , ( ( "env", "corollary" ), \x y -> SRH.setTheoremNumber x y )
        , ( ( "env", "equation" ), \x y -> SRH.setEquationNumber x y )
        , ( ( "env", "align" ), \x y -> SRH.setEquationNumber x y )
        , ( ( "smacro", "bibitem" ), \x y -> SRH.setBibItemXRef x y )
        ]
```

## Some conclusions

The reliance of `Accumulator.parse` and `Accumulator.render`
on the `LatexState` record, as well as the `latexStateReducerDict`
for dispatching calls to `latexStateReducer`, makes it very easy
to add new features, e.g., new macros and environments whose rendering requires a computed state. Moreover, it is
not hard to add fields to `LatexState` in order to add further new features.
Indeed, the first version of `LatexInfo` had only a `counters` field.
Others were added later as the scope of MiniLatex grew.
