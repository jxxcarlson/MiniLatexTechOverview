# List Machine

In this section we will discuss the `ListMachine`.
It is used to rationalize the space between
`LatexExpressions` in list of such values.
A `ListMachine` reads a tape from left to right.
The tape is aware of the contents of the current
cell, and the cell to the immediate left and immediate
right. Assume that cell contents have type `a`.
Then the machine has three registers, `before`, `current`,
and `after` which have type `Maybe a`. The use of `Maybe`
accounts for the fact that a register may not be defined
when the tape is near its beginning or its end. The main definition
is the followong

```
runMachine : (State a -> b) -> List a -> List b
runMachine outputFunction inputList =
  run (makeReducer outputFunction) inputList
```

Given a function `State a -> b`, one transforms
a list of `a`'s to a list of `b`'s. A value
of type `State a` holds the registers `before`, `current`,
and `after`, as well as an input tape of type `List a`:

```
type alias State a = {before: Maybe a, current: Maybe a, after: Maybe a, inputList: List a}
```

To use a `ListMachine`, it is enough to define

```
outputFunction: State a -> b
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
> import MiniLatex.Example exposing(..)
> runMachine sumState [0,1,2,3,4]
[1,3,6,9,7] : List Int
```

## The internals of runMachine

To begin, define and augmented state which
consists of a `State` field and the output list:

```
type alias TotalState a b = {state: State a, outputList: List b}
```

We will define an accumulator with this type:

```
TotalState a b -> List a -> TotalState a b
```

Following the path laid down before, one needs a function of type
`Reducer a b = (a -> b -> b)`. One uses it with a fold to
make an accumulator of type `TotalState a b -> List a -> TotalState a b`.

```
makeAccumulator : Reducer a b -> TotalState a b -> List a -> TotalState a b
makeAccumulator reducer initialMachineState_ inputList =
  List.foldl reducer initialMachineState_ inputList
```

It remains to prepare the initial state, apply the accumulator,
exract the output list, and reverse it to put it in proper form:

```
run : Reducer a b -> List a -> List b
run reducer inputList =
  let
    initialTotalState_ = initialTotalState inputList
    finalTotalState = (makeAccumulator reducer) initialTotalState_ inputList
  in
    List.reverse finalTotalState.outputList
```
