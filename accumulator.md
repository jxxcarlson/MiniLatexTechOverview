# Accumulators

As noted in the overview, one uses "Accumulators" to
collect and use information about sequentially numbered
sections, cross-references, tables of content, etc.
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
acc trasnformer state_ inputList =
  List.foldl transformer (state_, []) inputList
```

This function has type

```elm
Accumulator a b = state -> List a (state, List b)
```