# Accumulators

As noted in the overview, one uses "Accumulators" to
collect information about sequentially numbered
sections, cross-references, tables of content, etc.,
then use it to render the source text with these features.
Accumulators are made of up of reducers and folds:

```elm
type alias Reducer : a -> b -> b
List.foldl : (a -> b -> b ) -> b -> List a -> b
```

Thus a `Reducer` is just a name for the type of the
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

This function has type

```elm
Accumulator a b = state -> List a (state, List b)
```

## Accumulator.parse

Let us now discuss the accumulators used in MiniLatex.
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
parses the string and computes a new `LatexState`. Finally,
it returns a pair consisting of the new `LatexState` and
inputList with the parsedInput appended.

```
latexStateReducer : List LatexExpression -> LatexState -> LatexState
```

## Accumulator.render

```
render :
    (LatexState -> List LatexExpression -> a)
    -> LatexState
    -> List (List LatexExpression)
    -> ( LatexState, List a )
render renderer latexState paragraphs =
    paragraphs
        |> List.foldl (renderReducer renderer) ( latexState, [] )
```

```
renderReducer :
    (LatexState -> List LatexExpression -> a)
    -> List LatexExpression
    -> ( LatexState, List a )
    -> ( LatexState, List a )
renderReducer renderer input ( state, outputList ) =
    let
        newState =
            latexStateReducer input state

        renderedInput =
            renderer newState input
    in
    ( newState, outputList ++ [ renderedInput ] )
```

## LatexStateReducer

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
    latexStateReducerDispatcher ( theInfo.typ, theInfo.name ) theInfo latexState
```
