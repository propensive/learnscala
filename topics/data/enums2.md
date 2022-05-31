## Parameterized Enumerations

Enumerations provide a way of representing that a value may be in exactly one of a selection of possible states.
In addition to the simple enumerations we have already seen, where each of those possible states is either
selected, or it is not, we can also define that some or all of those states require additional parameters which
must be specified for the value to be in those states.

For example, we may use an enumeration to represent the arrival time of a flight, for the purpose of checking
the validitity of compensation claims. If the flight has not yet arrived, its arrival time will be scheduled,
and the scheduled arrival time will be known; if it arrived on-time or was canceled, then there is no further
information we need to record about the flight; and if the flight is late, we need to record how many minutes
late it is.

We can represent these four possibilities with an enumeration,
```scala
enum Arrival:
   case Scheduled(time: Time)
   case OnTime
   case Canceled
   case Late(minutes: Int)
```
and we can now call `OnTime` and `Canceled` _singleton cases_ to distinguish them from the
_parameterized cases_.

As before, an `Arrival` instance must be in exactly one of four possible states—scheduled, on-time, canceled or
late—but if scheduled, then it must be scheduled for a specified time, and if late, the number of minutes late
must also be specified.

Unlike simple enumeration cases, we are not permitted to write more than one parameterized case on the same
line, but we may still combine several singleton cases together, like so:

```scala
enum Arrival:
   case Scheduled(time: Time)
   case OnTime, Canceled
   case Late(minutes: Int)
```

Each of these four cases will also introduce a new type, which will be a subtype of `Arrival`. The parameterized
cases will introduce types `Arrival.Scheduled` and `Arrival.Late`, and the singleton cases will introduce
singleton types, `Arrival.OnTime.type` and `Arrival.Canceled.type`.

## Enumeration Methods

There are some limitations to the enumeration methods we have available for enumerations which have at least one
parameterized case. There would be no `Arrival.valueOf` or `Arrival.values` methods because some of the values
cannot be instantiated without their parameters (and there is no provision for supplying these). The method
`Arrival.fromOrdinal` _does_ exist, and integers will be assigned to cases in exactly the same way as they would
if the parameters did not exist, but a `NoSuchElementException` will be thrown when trying to access a
parameterized enumeration case by its index.

Nevertheless, the `ordinal` method of all every case will always return an integer value, and all cases have
`toString` methods which show not only the name of the enumeration case, but also their parameters (if they
exist).

For example, `Arrival.Late(140).toString` will produce the string, `"Late(140)"`.

## Type-Parameterized Enumerations

The types of the parameters of an enumeration do not need to be known concretely. It is possible to write an
`enum` template whose cases have parameters with types depending on a type parameter of the template.

Imagine that we wanted to introduce a type which represents a value and its validity. The value may be valid or
invalid, and in the latter case, we will include some information (a reason) why the value is not valid.

```scala
enum Validity[T]:
   case Valid(value: T)
   case Invalid(reason: String)
```

This would allow us to create new instances such as `Valid(Time(9, 27))` (representing `9.27am`) or
`Invalid[Time]("The minutes value is too high")` (representing the failure to verify a time), and these
instances would both have the type `Validity[Time]`, while we could use the same template to create instances of
other types like `Validity[String]` or `Validity[EmailAddress]`.

Of course, there is no reason why an `enum` template cannot include more than one type parameter.

## Type Parameters and Conformance

It may have been obvious, however, that constructing a new `Invalid[Time]` required us to explicitly specify
the `Time` parameter, when that was not necessary to construct the `Valid` case. That is because the parameter,
`Time(9, 27)` was sufficient to infer the type parameter as `Time`, and there was no equivalent parameter from
which to infer the type parameter for the `Invalid` case.

We can avoid this if the compiler _expects_ a particular type, like so,
```scala
def validation(value: Int): Validity[Int] =
   if check(value) then Valid(value) else Invalid("Failed validation")
```
or even when one branch of an expression suggests a type for a branch which has no information about that type,
as in this version which omits the return type:
```scala
def validation(value: Int) =
   if check(value) then Valid(value) else Invalid("Failed validation")
```

Here, Scala typechecks `Valid(value)` and infers its type as `Validity[Int]` because `value` has the type `Int`.
The other branch of the `if` expression returns a `Validity` instance, but without any constraint on its
parameter type. Though the type from the `then` branch is sufficient to cause `Invalid("Failed validation")` to
be instantiated as a `Validity[Time]`.

Enumerations may have type parameters, but by default, it is not possible for an enumeration to define type
parameters _and_ singleton cases.

Let's consider an enumeration type `Maybe`, similar to the familiar type `Option`,
```scala
enum Maybe[T]:
   case Just(value: T)
   case Empty
```
one of whose cases is a singleton case.

While the type of `Just('x')` will be a `Maybe[Char]` and `Just(12)` will be a `Maybe[Int]`, it is not possible
to define a type for `Empty` which is both a `Maybe[Char]` _and_ a `Maybe[Int]`.

The reason is that `Maybe[Char]` is a type which represents all those instances which were constructed from a
`Maybe` template with `T` equal to `Char`, while `Maybe[Int]` represents all those instances which were
constructed from the same template but with `T` equal to `Int`, and there is no solution for which `T` is
concurrently _equal_ to both `Char` and `Int`.

So the `Maybe` example above will not compile.

Scala's type system permits type parameters to be _variant_, but this is an advanced feature which will be
covered later in this course. In the meantime, an easier solution is to redefine `Empty` as a parameterized
case, with zero parameters, which we can write as follows:
```scala
enum Maybe[T]:
   case Just(value: T)
   case Empty()
```

This will now compile. The disadvantage with this approach is that multiple instances of the `Empty()` case
will exist, constructed with the template's `T` parameter equal to different types in different instances.

Despite being different instances, an `Empty[Int]()` will be equal to an `Empty[Char]()` because equality for
enumeration cases is defined on their case (in this example, `Empty` rather than `Just`) and their value
parameters, of which there are none.

We can learn later how to use variance in Scala to write an improved solution which can still use singleton
cases, without requiring multiple different instances. But for now, this approach will satisfy most purposes.
