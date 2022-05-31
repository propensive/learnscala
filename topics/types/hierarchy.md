## Type hierarchy diagrams

Subtyping relationships between some types in Scala imply there is a partial order between them. That is to say,
for any two concretely-known types, `A` and `B`, `A` may be a subtype of `B`, or `B` may be a subtype of `A`, or
there may be no subtyping relationship between them.

Subtyping is transitive, so if `A` is a subtype of `B` and `B` is a subtype of `C` then we can infer that `A` is
a subtype of `C`. If we know all the relationships between a set of types, we can draw the types in a hierarchy
to show these relationships.

```scala
sealed trait Fruit
sealed trait CitrusFruit extends Fruit, Edible
sealed trait Edible
sealed trait Berry extends Fruit

case class Apple() extends Fruit, Edible
case class Lemon() extends CitrusFruit
case class Orange() extends CitrusFruit
case class Poisonberry() extends Berry
case class Blackcurrant() extends Berry, Edible
```

![Class hierarchy diagram](/api/content/courseImages/scala/hierarchy.png)

We conventionally draw supertypes positioned above their subtypes, with arrows between them to indicate direct subtyping
relationships. Conventionally, again, the arrows point _from_ a subtype _to_ its supertype, so they are always directed
upwards. There are a few reasons we could use to justify this convention, but it's probably easiest to remember that the
a subtype needs to refer to (or point to) its supertype for some of its definitions, but a supertype can happily exist
without any reference to its subtypes.

Diagrams like these can help to more clearly communicate subtyping relationships. For example, if one type
appears visibly lower than another in the diagram, then (without even studying the arrows) it cannot be a
supertype; if types appear side-by-side, then they do not have any subtyping relationship between them; if we
can find a path by following the arrows through a series of types, we know that they do have a _subtyping_
relationship, and it is usually easier to see than it would be by reading the class definitions.

## The Universal Hierarchy

All types in Scala belong to a universal _type hierarchy_, though as the number of possible types Scala can
encode is infinite—we will discover later how new types can be constructed by combining existing types—the
hierarchy would also be infinite. And that would obviously be impossible to draw! But we can draw the part of
the hierarchy that we are interested in, overlooking the parts we don't care about, and it can remain a useful
diagram.

And while the total number of possible types that Scala can represent is infinite, we will only ever work with a
finite number, and furthermore, the full inheritance lineage for any type will be finite; that is to say, for
every type, if we follow the arrow to its supertypes, then take the supertypes of its supertypes, and carry on
_upwards_ like this, we can count the steps before we reach the "top", and it would be unusual for it to be more
than a few.

## `Any`, the Top Type

But what is the "top", and why should we stop there? Scala's type hierarchy has a "top type" whose name is
`Any`. `Any` is a supertype of every other type. That means that for every value we encounter, whatever type it
has, its type will be a subtype of `Any`, and a field of type `Any` can hold any value.

```scala
val x: Any = 1
val y: Any = "two"
val z: Option[Any] = Some(Exception("three"))
```

`Int`s, `String`s and `Exception`s are all subtypes of `Any`, which means that `Any` represents the small set of
properties that's common to all of these values: the set of things that can be done to all objects, such as
checking equality against another value, getting a deterministic (but theoretically arbitrary) hash code
corresponding to the object, getting a value which represents the value's runtime type, viewing a representation
of the object as a `String`, and some other methods we will probably never need to use.

So, for the values, `x` and `y` above, which both have the type `Any`, despite them being (at runtime) an `Int`
and a `String` respectively, we cannot use any properties like `+` or `substring` because the type `Any` limits
us, at compiletime, to the properties that are common to all values.

`Any` does not have any supertypes (at least, it has none that we need to consider right now!), because there
would be no need for them to exist: every value is an instance of `Any`, and any supertype must be a superset
of the instances of `Any`, but our set is already _everything_, so any superset, if it were to exist, could only
be the same superset.

## `AnyRef` and `AnyVal`

`Any` has two immediate subtypes, `AnyRef` and `AnyVal` and all values will be partitioned into these. Most
values we work with will be `AnyRef`s: every instance of every class, enum or case class, every `String`, every
function, every `Array` and every singleton object. Collectively, these are all _reference types_, and they are,
by definition, subtypes of `AnyRef`.

The types corresponding to any class or object which we define will automatically be subtypes of `AnyRef`,
without needing to specify that fact explicitly.

What remains, are the _primitive types_, and types derived from them. There are a fixed number of primitive
types:
- `Byte`, `Short`, `Int` and `Long`, types representing signed integers
- `Float` and `Double`, types representing floating-point numbers
- `Char`, for Unicode characters, and
- `Boolean` for `true` and `false` values.
- `Unit`, the type with only one value, `()`

These nine types all have native representations on the JVM, and may be manipulated directly through bytecode.
We cannot define new primitive types. (Doing so would require us to "invent" new bytecode instructions to
operate on them, too!) But Scala does currently allow us to define new _value classes_: types which may be based
on a primitive type, while remaining distinct from that type, and these, plus the primitive types themselves,
are all the subtypes of `AnyVal`.

So together, `AnyVal` and `AnyRef` cover all values in Scala. While most types we use in Scala have no special
status in the _language_ (though may be used heavily by libraries, including the standard library), `Any`,
`AnyVal` and `AnyRef` are inherent to the language. And there are a few additional types in the Scala type
hierarchy which the compiler treats specially.

These include the types `Product` and `Serializable`, which are automatically supertypes of every case class.
The type `Tuple` is a supertype of every tuple type. And `Singleton` is automatically a supertype of every
singleton type.

And while we usually think of types as being defined in terms of their supertypes—types, derived from their
class definitions, where those classes _inherit_ from other classes—there are two "magic" types which exist that
are automatically subtypes of every new class.

## `Nothing` and `Null`

The `Nothing` type, sometimes referred to as the _bottom type_ is the counterpart to `Any`, and appears at the
bottom of the type hierarchy, being a subtype of _everything_. If we could ever have an instance of `Nothing`,
it would be an extremely useful value, because it would have _all_ the properties of _every_ other type! But
such a value cannot possibly exist, so `Nothing` is a type with _zero_ instances: it's not possible to create
one or return one. In fact, a method which declares a return type of `Nothing` can only ever throw an exception,
or call a method that returns `Nothing` (such as itself), which can ultimately only ever throw an exception. But
the type nevertheless exists to complete the hierarchy.

Here is a comparison of `Nothing` and `Any`:

|                             | `Nothing` | `Any`     |
|-----------------------------|-----------|-----------|
| instances                   | none      | all       |
| properties                  | all       | very few  |
| position in hierarchy       | bottom    | top       |
| relationship to other types | subtype   | supertype |

In addition to `Nothing`, which has no values, there is another type called `Null` which has just a single
value, the value `null`. `Null` is, of course, a supertype of `Nothing` (because every type is), but is a
_subtype_ of every _reference type_, that is, every subtype of `AnyRef`. While it's almost never desirable to
use it, the `null` value, which exists as a uniquely-special value on the heap (there is only one universal
`null` value), is a valid instance for every reference type. The value `null` has the type `Null`, and by virtue
of `Null` being a subtype of every other reference type, `null` is therefore an acceptable value for any
reference type. There is no equivalent subtype of all `AnyVal` types, as it is not useful in practice.

The Scala compiler may also be run with an experimental configuration parameter which removes the `Null` type
from its usual position in the type hierarchy. This is a stricter variant of the type hierarchy, and has some
implications for how we would write our programs with the feature enabled. We will explore this in a later
lesson.

All the interesting and useful types exist in the middle of the hierarchy, between `AnyVal` and `AnyRef` at the
top, and `Nothing` at the bottom. Usually we do not want to work with types like `Any`, `AnyVal`, `AnyRef`,
`Null` and `Nothing` because they either represent too many values to be useful, or too few values to be useful.
But they nevertheless exist, and can be shown to exist, and serve to complete Scala's type hierarchy.
Occasionally, we will encounter them, and it's necessary to understand the role they serve as types.
