# List Machine

In this section we will discuss the `ListMachine`.
It is used to rationalize the space between
`LatexExpressions` when they are rendered.
A `ListMachine` reads a tape from left to right.
The tape is aware of the contents of the current
cell, and the cell to the immediate left and immediate
right. Assume that cell contents have type `a`.
Then the machine has three registers, `before`, `current`,
and `after`, all of which have type `Maybe a`. The use of `Maybe`
accounts for the fact that a register may not be defined
when the tape is near its beginning or its end. The main definition
is the following

```
ListMachine.run : (State a -> b) -> List a -> List b
ListMachine.run outputFunction inputList =
  run (makeReducer outputFunction) inputList
```

Given a function `State a -> b`, one transforms
a list of `a`'s to a list of `b`'s. A value
of type `State a` holds the registers `before`, `current`,
and `after`, as well as an input tape of type `List a`:

```
type alias State a = {before: Maybe a, current: Maybe a, after: Maybe a, inputList: List a}
```

## Example

Make the following file:

```
-- File : Example.elm

module MiniLatex.Example exposing(..)

import MiniLatex.ListMachine exposing(State)

sumState : State Int -> Int
sumState internalState =
  let
    a = internalState.before  |> Maybe.withDefault 0
    b = internalState.current |> Maybe.withDefault 0
    c = internalState.after   |> Maybe.withDefault 0
  in
    a + b + c
```

Then do this:

```
> import MiniLatex.ListMachine as ListMachine
> import MiniLatex.Example exposing(..)
> ListMachine.run sumState [0,1,2,3,4]
[1,3,6,9,7] : List Int
```

## The internals of ListMachine

To use the ListMachine module, it is enough to export the type
`State a`, the function `ListMachine.run`, and to define an
output function `f : State a -> a`. The code for
all this on [GitHub](https://github.com/jxxcarlson/meenylatex/blob/master/src/MiniLatex/ListMachine.elm).
Here we describe brielfly the design of this module.
There is a fair amount of internal plumbing consisting
of short type and function definitions. The idea is
to hide all implementation details behind a very simple API.
Here is the definition of `run`:

```
ListMachine.run : (State a -> b) -> List a -> List b
ListMachine.run outputFunction inputList =
  run_ (makeReducer outputFunction) inputList
```

The function `makeReducer` transforms a function of
type `State a -> b` to a function of type

```
Reducer a b : a -> TotalState a b -> TotalState a b
```

where `type alias TotalState a b = {state: State a, outputList: List b}`.
Next, we construct a function that returns
a function of type `TotalState a b -> List a -> TotalState a b`
given a `Reducer a b`:

```
makeMachine : Reducer a b -> TotalState a b -> List a -> TotalState a b
makeMachine reducer initialMachineState_ inputList =
  List.foldl reducer initialMachineState_ inputList
```

With this function in hand, we can define `run_`:

```
run_ : Reducer a b -> List a -> List b
run_ reducer inputList =
  let
    initialTotalState_ = initialTotalState inputList
    finalTotalState = (makeMachine reducer) initialTotalState_ inputList
  in
    List.reverse finalTotalState.outputList
```

For the remaining details, please see the
[code](https://github.com/jxxcarlson/meenylatex/blob/master/src/MiniLatex/ListMachine.elm).

## addSpace

The `Render2.addSpace` function is the output function for
a `ListMachine` with type `State LatexExpression`. Thus
it transforms a tape of `LatexExpressions` into another
tape of the same kind. The output function given below
defines the rules for adding space before or after
a `LXString String`, or not adding any space at all:

```
addSpace : ListMachine.State LatexExpression -> LatexExpression
addSpace internalState =
    let
        a =
            internalState.before |> Maybe.withDefault (LXString "")

        b =
            internalState.current |> Maybe.withDefault (LXString "")

        c =
            internalState.after |> Maybe.withDefault (LXString "")
    in
    case ( a, b, c ) of
        ( Macro _ _ _, LXString str, _ ) ->
            if List.member (firstChar str) [ ".", ",", "?", "!", ";", ":" ] then
                LXString str

            else
                LXString (" " ++ str)

        ( InlineMath _, LXString str, _ ) ->
            if List.member (firstChar str) [ "-", ".", ",", "?", "!", ";", ":" ] then
                LXString str

            else
                LXString (" " ++ str)

        ( _, LXString str, _ ) ->
            if List.member (lastChar str) [ ")", ".", ",", "?", "!", ";", ":" ] then
                LXString (str ++ " ")

            else
                LXString str

        ( _, _, _ ) ->
            b
```

Here are the two small auxilliary functions:

```
lastChar =
    String.right 1


firstChar =
    String.left 1
```
