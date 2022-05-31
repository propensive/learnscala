## Dot-Notation

In object-oriented programming, the "dot-notation" which we use to access the members of an object is a
fundamental part of how we think about objects and their members. A path to an object written in dot-notation
can be thought of as an address for accessing a particular object in memory.

For example,
```scala
country.city.street
```
would access the `city` member object of `country`, and then the `street` member of that `city`.

And, of course, this syntax corresponds directly to the structure of the `country`, `city` and `street` objects
as they exist in memory, each `.` choosing the field of the object containing the reference we should follow,
and is often known as _dereferencing_.

And since the type of each object is known statically, the Scala compiler can enforce (at compiletime) the
restriction that only members within an object that are known to exist at runtime may be dereferenced.

And the correspondence between dot-notation in syntax and the structure of objects in memory is normally so
tight that it may seem surprising to discover that Scala allows a more versatile interpretation of the dot (`.`)
in dot-notation.

This feature is unexpected enough that it must be first enabled with a language import,
```scala
import scala.language.dynamics
```
and all the examples in this lesson assume the import is in scope.

## The `Dynamic` trait

The trait, `scala.Dynamic` (or simply `Dynamic`, since everything in the `scala` package is always in scope), is
a marker trait—a trait with no methods or members of its own, but which the compiler is specially aware of—that
can relax many of the static constraints on dereferencing for any object which inherits from it.

The simplest example of a `Dynamic` object would allow a member with _any name at all_ to be accessed by
dot-notation.

So, given an instance, `pool`, of a class, `Pool`, which is a subtype of `Dynamic`, we may call `pool.xyz` or
`pool.data` or `pool.someMember`, without the author of `Pool` having even considered `xyz`, `data` or
`someMember` as members of the class `Pool`. And no compile errors will be produced!

## Rewriting

Of course, any code which the compiler _accepts_ as valid must compile to _something_. And bytecode which
attempts to access nonexistent members (either methods or fields) would not only be unacceptable to the JVM, but
would also be useless! What purpose could it possibly serve?

So instead, Scala transforms any attempt to access a nonexistent member of a `Dynamic` object into a call to a
method which _does_ exist, converting the nonexistent member name into a `String` and passing it as a parameter
instead.

For the simplest member accesses, the conversion produces a call to a method called `selectDynamic`.

For example, the expression,
```scala
pool.xyz
```
would be transformed into:
```scala
pool.selectDynamic("xyz")
```

This does, however, require that the method `selectDynamic` _is_ defined on the object `pool`, and that it can
accept a `String` as its only parameter.

The `selectDynamic` method is not part of an object-oriented interface, though, since `Dynamic` is only a marker
trait, and its precise signature may vary in a number of ways. It could take type parameters, or given context.
Its parameter could be typed as `Any`, or it could be defined to take repeated arguments or default arguments,
even if it is always called with just one argument—the apparent member name.

It could even be implemented as an inner object with an apply method (taking a `String`)! The key requirement is
that the method `selectDynamic` may be called with a single `String` parameter, and this becomes clear if we
attempt to access a nonexistent member of a `Dynamic` object which is missing a suitable `selectDynamic`
implementation.

Consider the class,
```scala
class Container() extends Dynamic
```
which is `Dynamic`, but has no `selectDynamic` method, with the attempted invocation:
```scala
Container().missing
```

The compiler will identify this as an error; not the error that `missing` is not a member of `Container`, but
instead:
```
value selectDynamic is not a member of Container
possible cause: maybe a wrong Dynamic method signature?
```

This message makes it a little more obvious that the compiler has attempted a transformation to a `Dynamic`
signature, and which method is involved in that transformation.

## Other Transformations

Different types of expression involving dot-notation on `Dynamic` objects (in all the examples below, called
`value`) will be rewritten by the compiler into statically-typed expressions involving the real methods,
`selectDynamic`, `applyDynamic`, `applyDynamicNamed` and `updateDynamic`.

These transformations include a variety of different types of expression, including not just member access, but
method invocations with multiple parameters and parameter lists, with named parameters.

The following table describes these transformations.

Original expression                  | Transformed expression
-------------------------------------|-------------------------------------------------------------------
`value.member`                       | `value.selectDynamic("member")`
`value.member(param) = newValue`     | `value.selectDynamic("member").update(param, newValue)`
`value.member(param1, param2)`       | `value.applyDynamic("member")(param1, param2)`
`value.member(param1)(param2)`       | `value.applyDynamic("member")(param1).apply(param2)`
`value.member(name = param)`         | `value.applyDynamicNamed("member")(("name", param))`
`value.member(name = param, param2)` | `value.applyDynamicNamed("member")(("name", param), ("", param2))`
`value.member = newValue`            | `value.updateDynamic("member")(newValue)`

Expressions of the form shown on the left of this table will be transformed into the form shown on the right, as
long as the object `value` is known statically to be a subtype of `Dynamic`, and as long as they would not
compile without the transformation.

So calling `value.member` on a `Dynamic` object `value` which has a real field whose name is `member` would
just return the field's value (as always), without attempting to rewrite to `value.selectDynamic("member")`.

We should be particularly aware of any methods that are defined on the `Any` type (such as `clone` and `wait`)
since these will always be members of every object, and thus can never be transformed into `Dynamic`
invocations. Some of these are easily forgotten!

And in general, the implementations of `selectDynamic`, `applyDynamic`, `applyDynamicNamed` and `updateDynamic`
need only be such that the transformed expressions on the right side of the table will compile, so parameter
types and return types may be chosen accordingly.

These transformations have a couple of interesting curiosities.

The expression, `a.b(c)`, on a `Dynamic` value, `a`, will _not_ transform to `a.selectDynamic("b")(c)` or
`a.selectDynamic("b").apply(c)`, even though these would also be reasonable transformations. But the distinction
does make it possible for the code,
```scala
val b = a.b
b(c)
```
to have different behavior to,
```scala
a.b(c)
```
as the latter is transformed as an entire expression to `applyDynamic`.

`Dynamic` expressions with multiple parameters or parameter blocks generally transform in predictable ways, but
some care should be taken when one or more parameters in an expression are _named parameters_, as the
transformed invocation may change from `applyDynamic` to `applyDynamicNamed`, which could have entirely
different behavior.

A quirk of `applyDynamicNamed` is that any _unnamed_ parameter passed to it will transform to a two-tuple whose
first value will be the empty string, `""`. It is common for `applyDynamicNamed` to take repeated arguments, but
more restrictive implementations which take a fixed number of arguments are possible.

It is also uniquely possible with method calls that transform into `applyDynamicNamed` invocations to pass more
than one named parameter with the same name, since these simply transform into additional _positional_
arguments!

## Combining with Singleton Literals

While there are times when it may be useful to have values upon which a member with _any name_ may be accessed
using dot-notation, that will very frequently offer users _too much_ flexibility. And using a type of `String`
for the named "member" of a `Dynamic` object suggests a less permissive possibility using singleton literal
types, and union types.

Each of the special `Dynamic` methods takes, as the first parameter in its first parameter block, the name of
the apparent member being accessed. So, in each definition, this parameter may be more restrictively typed to
permit only certain string values.

For example, we could use the union of singleton types, `"sum" | "product"`, to restrict calls to an
`applyDynamic` method, as follows:
```scala
object Calc extends Dynamic:
  def applyDynamic(method: "sum" | "product")(xs: Int*): Int =
    if method == "sum" then xs.sum else xs.foldLeft(1)(_*_)
```

This permits calls to `Calc.sum(1, 2, 4)` or `Calc.product(1, 2, 4)`, which would produce the results `7` and
`8` respectively, while forbidding a method with any other name to be invoked on `Calc` (except those inherited
from `Any`, of course). Only at runtime, by checking the value of the `method` parameter, is the particular
implementation to be used chosen.

In a more advanced example, the singleton types could be provided through a type parameter.

## Limitations

While an object which extends `Dynamic` will _seem_ to have a variety of apparent members, accessible through
familiar dot-notation, from the type system's perspective (and indeed, in reality) these members do not exist,
except as syntactic rewrites.

A consequence of this is that a `Dynamic` object will never conform to an unrelated template interface, even if
it appears to conform through dynamic method invocations. For example, given the definitions,
```scala
trait StringCell:
  def value: String

def access(cell: StringCell): String = cell.value
```
and a `Dynamic` object,
```scala
object HelloWorld extends Dynamic:
  def selectDynamic(method: "value"): String = "Hello World!"
```
upon which it is certainly _possible_ to call `HelloWorld.value`, we are nonetheless forbidden from calling,
`access(HelloWorld)`.

Similarly, it's not possible to call methods on a generic `Dynamic` instance without knowing the specifics of
how its dynamic methods are implemented, since their signatures are needed for typechecking.

So a method definition such as,
```scala
def access(cell: Dynamic) = cell.value
```
would fail to compile since the type of `cell` is not known to have a `selectDynamic` method, or what its return
type would be.

Additionally, the erasure of Scala's types means that it is not possible to overload many singleton-typed
variants of dynamic methods, unless their erased signatures can somehow be disambiguated, since any `String`
literal singleton type will erase to `String`.

This means that it's possible to define both,
```scala
def applyDynamic(m1: "alpha")(): Unit = ()
def applyDynamic(m2: "beta")(value: Int): Unit = ()
```
but only by virtue of their arity providing a way to disambiguate between them, and,
```scala
def applyDynamic(m1: "alpha")(): Unit = ()
def applyDynamic(m2: "beta")(): Unit = ()
```
could not be defined on the same object, as the types `"alpha"` and `"beta"` both erase to the same type,
`java.lang.String`.

## A Word of Caution

The `Dynamic` trait offers a lot of flexibility to reinterpret the familiar syntax of dot-notation in custom
ways. But it should be used sparingly.

Scala developers are not only very familiar with dot-notation, but also very familiar with its _usual_
interpretation. And the use of `Dynamic` introduces new, custom, and potentially unexpected interpretations that
risk surprising users, and undermining their confidence in the language.

That is not to say that `Dynamic` should _never_ be used, but its benefits should always be weighed against its
potential to confuse.
