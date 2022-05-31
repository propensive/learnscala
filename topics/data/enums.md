## Simple Enumeration Definitions

We often need to work with data which must be in exactly one of a number of different possible states.
_Enumerations_, which are introduced by the `enum` keyword in Scala offer a way to represent such data.

We can define an enumeration like this:

```scala
enum Activity:
   case Active, Dormant, Extinct
```

This introduces a new type, called `Activity`, which has three instances, `Active`, `Dormant` and `Extinct`
representing the three possible states of a volcano may be in. In our model, it can't be in a state that is not one of
these three possibilities, and it cannot be in _more than one_ of these states at the same time.

It's also possible to write this enumeration more verbosely with multiple `case` lines, like so:

```scala
enum Activity:
   case Active
   case Dormant
   case Extinct
```

This style is more commonly used for more complex enumerations, which are covered in the next lesson.

An enumeration is a simple example of a _sum type_ or a _coproduct_ (which are names that arise in contrast to
_product types_), though there are other more general ways of representing these in Scala, so we will always call them
enumerations when they are defined this way.

The primitive type, `Boolean`, which is built into the language, already behaves like an enumeration: a
`Boolean` value is either `true` or `false`. And other primitive types do, as well, though we wouldn't normally consider
them as such due to the number of possible values they can represent. For the `Long` type, there are eighteen
quintillion different possible values, while for an enumeration, we are normally limited to just a handful: usually
fewer than ten, and never more than about twenty unless there is some exceptional justification for it.

By default, an enumeration's cases are accessed through the _enumeration object_, a singleton object which exists by
virtue of the `enum` definition. So we would need to refer to `Activity.Dormant` or `Activity.Active`
in general. But it would not be unusual to import these values into a scope to make the code less verbose, like so,

```scala
import Activity.*
```

or more explicitly,

```scala
import Activity.{Active, Dormant, Extinct}
```

Unlike a class or a trait, it's not possible to add instances to an enumeration later _in dependent code_
(though we can always modify an `enum` definition later _in time_, of course)! But knowing that the set of possible
enumeration instances is fixed means that code can exploit this fact to know that a pattern match is exhaustive.

In the following example, the Scala compiler knows that we have specified an action for every possible
`Activity`,

```
def action(state: Activity): Unit = state match
   case Active  => leaveImmediately()
   case Dormant => ensureReadyToLeave()
   case Extinct => ()
```

and that we do not need to consider a case where the `state` is something else.

## Enumeration Methods

An `enum` definition can also include the same definitions as an `object` definition, but unlike an object, these
members are available on each instance of the enumeration. The value `this` is also available inside an
`enum` body, and may be used in method definitions to provide behavior which depends on the particular enumeration case.

Here is an example of an enumeration of the days of the week with a `Boolean` method to determine if a particular `Day`
case represents a weekend.

```scala
enum Day:
   case Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday

   def weekend: Boolean = this match
      case Saturday | Sunday => true
      case _                 => false
```

We can then call `Saturday.weekend` (which returns `true`) or call `day.weekend` where `day` is a `Day`
instance. But there is no method, `Day.weekend`.

## Representation

An enumeration provides us with a convenient, opaque and domain-specific programmatic interface for working with
different values which are internally represented by integers. The name "enumeration" is derived from the fact that the
cases of the enumeration can be counted. And we could _carefully_ write the same programs with enumerations or with a
set of integers, for example by representing,

- `Active` by `0`,
- `Dormant` by `1`, and,
- `Extinct` by `2`.

And that would be much closer to the Java bytecode that Scala will produce when we write `enum` definitions.

But we could say that the advantages that enumerations provide over integers are almost _innumerable_!

While `Active` and `Dormant` can be as easily distinguished from each other as `0` and `1`, we would not be able to
distinguish between `Active` and `Monday` (from two completely unrelated enumerations) if both were represented by the
integer, `0`.

Integers also have a much wider range of values. An `Int` can be `3` or `4` even though there are no
corresponding `Activity` values for these integers, so there is no protection against representing _impossible_
states.

And while equality and disambiguation are useful properties of both integers and enumeration cases, arithmetic
operations are useful _only_ for integers: it would make no sense to add `Tuesday` and `Thursday` and get
`Friday`.

Integers are also not resilient to refactorings where the set of enumeration cases changes. If we were to add
an `Erupting` case to our `Activity` enumeration between `Active` and `Dormant`, then the representation of
`Dormant` would be renumbered from `1` to `2`, and `Extinct` from `2` to `3`, but our code would not need to change as a
consequence. But had we used integers, then every usage would also need to change.

Some programs will use constant aliases, such as,

```scala
final val Active: Int = 0
final val Dormant: Int = 1
final val Extinct: Int = 2
```

but it only solves some of the issues.

So enumerations provide us with types with very useful constraints over `Int`s regarding what operations we should and
should not be able to perform on them.

## Converting Enumerations

There are, however, some times when it is useful to _see_ the numerical representation of an enumeration. We can access
this using any enumeration's `ordinal` method. For example, assuming the definition of `Day`, we could obtain `Friday`'s
ordinal value with,

```scala
val fridayValue: Int = Friday.ordinal
```

which would be the integer `4`, as `Friday` is the fifth case listed, and numbering (as usual) starts from `0`
in Scala.

Conversely, if we have an integer, we can find its corresponding enumeration value using the `fromOrdinal`
method of the enumeration object, in this case, `Day`:

```scala
val day: Day = Day.fromOrdinal(4)
```

There is nothing to stop us calling `Day.fromOrdinal(100)`, which has no corresponding `Day` value, so an exception will
be thrown in these cases.

More useful, in many ways, are two other complementary methods for enumerations, `toString` on each enumeration
instance, and `valueOf` on the enumeration object. These behave in the same way as `ordinal` and `fromOrdinal`, except
that they return and take `String`s instead of `Int`s, where those `String`s are the names of the enumeration cases, as
defined in the code.

So, for example, `Saturday.toString` will produce the `String`, `"Saturday"`, and `Day.valueOf("Tuesday")` will return
the value `Tuesday`.

Finally, it is often useful to see all the enumerated cases of an enumeration in a collection. This is provided with
the `values` method of the enumeration object, which returns an `Array` of the cases.

For example, `Activity.values` returns `Array(Active, Dormant, Extinct)`.

All together, Scala provides very simple facilities for modeling and working with enumerated values. In the next lesson,
we will see how this can be extended to more complex data structures.
