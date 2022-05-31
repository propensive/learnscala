## Partial and Total matching

A common thought when writing a pattern match is knowing whether we are handling all possible inputs.

Conversely, we could ask, is it possible to provide an input to the pattern match expression which would match
_none_ of the patterns? This is not usually something we would want to happen: it would mean that the expression
is _partial_, not _total_. If we were to pass a scrutinee value to the pattern match which it could not handle,
in the absence of any possible return expression, the only action that makes sense is for an exception to be thrown.
So, Scala will throw a `MatchError`.

## Wildcard matches

To avoid _partiality_ (when an expression is not total) in the pattern match, one solution is to use a
_wildcard_ pattern. This is not really a "pattern" at all, because it is guaranteed to match the scrutinee every
time, but that's exactly what we need to ensure _totality_.

In it's simplest form, a wildcard match is the pattern, `_`. It must be the last case specified in a pattern
match; if we know it will match everything, then there would not be any point attempting to match on something
else _after_ a wildcard, and, in fact, the compiler considers it an error.
```scala
def chooseTheme(background: Color): Theme =
   background match
      case Black => Theme.Light
      case White => Theme.Dark
      case _     => Theme.Adaptive
```

The `_` pattern is useful if we don't care to _use_ the value we have matched in the case clause, but if we do,
then we can use a new identifier in place of the `_`, for example,
```scala
def message(count: Int): String = count match
   case 1 => "There is one."
   case 2 => "There are two."
   case 3 => "There are three."
   case n => s"There are $n."
```

Here, we introduce a new identifier, `n`, which is bound to the scrutinee after it has failed to match `1`, `2`
or `3`. This is something we have seen already, in fact. When we wrote the pattern, earlier,
```scala
case Color(red, green, blue) =>
```
the identifiers, `red`, `green` and `blue` are also patterns which are used to "match" against the three
parameters of `Color`, but being wildcard patterns, we know they will always match. The difference in this case
is that they always match on each of the parameters, rather than always matching the entire scrutinee value. So
their primary function is to bind new identifiers to each of the three parameters so we can use them directly on
the right-hand side of the case clause.

A case-class extractor such as `case Color(red, green, blue)` is guaranteed to match any instance of `Color`,
but if we had written, `case Color(0.0, green, blue)` then this would be known not to exhaustively match every
possible instance of `Color`—a scrutinee of `Color(1.0, 2.0, 3.0)` would not match, for example—and further
cases would need to be specified to ensure totality.

This applies recursively to nested case-class matches, of course.

## Enumerations and Sealed types

A wildcard match does not always make sense, though. If we were to to match a scrutinee which was an
`Option[Int]`, we would know that matching `case Some(value)` and `case None` would be guaranteed to match.
`Option[T]` is a _sealed type_, which means there are no other possible subtypes other than these two, and
therefore no need to provide a wildcard match at the end to handle any other cases.

This is something Scala is also able to know, as long as the type of the scrutinee is a union type, a sealed
type such as an enumeration, a `Boolean` or, trivially, a `Unit`. This feature is called
_exhaustivity checking_, and we say that it checks that the match is _exhaustive_ in the sense that it
_exhausts_ the set of possible values to consider such that there is nothing remaining to try to match against.
And it guarantees the _totality_ of the match expression, which is something we want.

Provided Scala can prove that, collectively, the cases listed inside a `match` will handle any value we pass to
it, it will consider the match exhaustive.

## Guards

Sometimes an extractor or the scrutinee's type don't give us enough ways to express exactly which cases we would
like to match, and which we wouldn't. Scala allows additional checks to be performed on the scrutinee, _after_
extraction and typechecking, but _before_ agreeing to match the case. These are called _guards_ and we can think
of them as "guarding" the execution on the right-hand side of the case clause; a final check "at the door". The
syntax for guards reuses the `if` keyword with a predicate between the pattern and the `=>` symbol, like so,
```scala
def howMany(value: Int): String = value match
   case 1           => "one"
   case 2           => "two"
   case n if n < 10 => "a few"
   case n           => n.toString
```

The match expression will check first if the scrutinee is equal to `1`, then if it is equal to `2`, then if it
is less than `10`, and if none of these cases match, the scrutinee is bound to the identifier `n`, and is
converted to a `String`.

Remember that the order of the cases is important. If we were to shuffle the order they appear in the list, it
could change the behavior of the match.

## Checking Exhaustivity

For each case in a match, Scala will ask itself a question: do there exist any instances of the scrutinee which
earlier cases would not match, but which this case would match? And for the entire match, it asks, are there
any instances of the scrutinee which none of the cases would match, and if so, what are they?

Based on the answers to these questions, Scala may issue warnings about the code. If it decides that a case
will never match because one of the earlier cases would always match it first, then we will receive a warning
that it is an _unreachable case_, meaning that it is provably impossible that it will never match. Likewise, if
it is possible for a particular scrutinee instance to pass over every case in the expression without matching
any of them, then a warning is issued: "match may not be exhaustive". We even get insight into the cases which
are not handled.

Remember that these questions are asked with the static information Scala has available at compiletime. It
knows nothing about the runtime value of the scrutinee, apart from its type. Scala is also limited in its
ability to analyse the predicates in guards: a predicate could be _any_ expression which returns a `Boolean`
value, even a nondeterministic expression, so the compiler makes no assumptions about whether a guarded case
will always match, never match or sometimes match, even when it might be obvious to us.
