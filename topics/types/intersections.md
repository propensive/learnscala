A type such as `Int` or `Ordering[Double]` can be thought about in two ways, simultaneously. A type can be seen
as a set of _properties_ which are provably true about a certain set of _instances_ of that type. Or it can be
seen as the set of _instances_ about which a certain set of _properties_ are provably true.

Each perspective is equally valid, and they are just different sides of the same coin—and this itself is a
reflection of the duality of the concepts of properties and instances—though sometimes one view is more useful
than the other for understanding a particular type-related idea.

Developers more often tend to think of types as sets of instances than sets of properties. And it is from this
angle that the name _intersection type_ makes most sense, but it is the type's properties that are most helpful
in understanding them, and therefore we will approach them in terms of their properties.

Intersection types occur very frequently in Scala, since they arise from calculating the _greatest lower bound_
(GLB) of two types. We will explore GLBs later, after we have developed an understanding of intersection types
themselves.

An intersection type, such as `Movable & Doubleclickable`, is the combination of two or more other types—in this
case, `Movable` and `Doublclickable`—using n ampersand: the symbol, `&`. An intersection type represents all the
properties from all of the types which make up the intersection. So `Movable & Doubleclickable` has every
property that `Movable` has, and every property that `Doublclickable` has.

## Relationship with Templates

If we were to define trait templates for `Movable` and `Doubleclickable`, like so,
```scala
trait Movable:
  def moveTo(position: Position): Unit

trait Doubleclickable extends Clickable:
  def doubleclick(): Unit
```
then an instance of `Movable & Doubleclickable` would have the properties,
```scala
def moveTo(position: Position): Unit
def doubleclick(): Unit
```
plus any that it inherits from `Clickable`.

An intersection types may be used anywhere, without any need for a corresponding class or trait template to be
created first. So while this exact list of properties may be represented by the type,
`Movable & Doublelickable`, we do not need to define a template in order to use the type.

So, we can define a method such as,
```scala
def change(item: Movable & Doubleclickable, position: Position): Unit =
  item.moveTo(position)
  item.doubleclick()
```
with no more than the definitions of `trait Movable` and `trait Doubleclickable`.

However, methods are defined so that they may be called! So in order for such a method definition to be useful,
an instance of `Movable & Doubleclickable` must, at some point, be constructed so that it may be passed to the
method, `change`.

Such an instance may be constructed from a template which inherits from both traits (either directly, or
indirectly via another trait), and implements all of its properties, such as:
```scala
class Window() extends Movable, Doubleclickable:
  def moveTo(position: Position): Unit = ???
  def doubleclick(): Unit = ???
  def click() = ??? // from Clickable
```

But the method `change` does not need to know about the existence of `Window`, nor would it need to know of some
other class called `Icon` which also implements `Movable` and `Doubleclickable`. It is defined only in terms of
the traits `Movable` and `Doubleclickable`, which serve as proxies for the properties defined within each.

### More Complex Intersections

When the properties of two types in an intersection are distinct—in the simplest sense, that every member either
exists with the same name and signature in both types, or exists uniquely in just one of the two types—their
intersection is as simple as collecting together the properties of both into the same type.

But the situation is more complex when properties with the same name but different signatures exist in both
types.

It is _always_ possible to construct an intersection type, even when this is the case, though the signatures of
members in the resultant intersection type may need to change to accommodate both types in the intersection.

While possible, this is nontrivial, and we will cover it in a later lesson.

However, from a practical perspective, there are no restrictions on using intersections of any other valid
types. Yet that does not guarantee that it is possible to construct an instance of such a type.

As a trivial example of this, we can use the type `Int & String` without error, even though we know that no
instances of such a type can exist: a value is either an `Int`, a `String`—or neither.

For example, the method definition,
```scala
def uncallable(value: Int & String): Unit = ()
```
is valid and will raise no error. But it is not useful.

### Equivalence by Properties

There are many such "impossible" or _uninhabited_ types which have zero instances (and can even be proven by the
compiler to have zero instances).

If types were just sets of instances, then it would be reasonable to consider two uninhabited types as equal to
each other. But they are not! Types are also sets of properties, and `Int & String` represents a different set
of properties to, for example, `Double & Ordering[String]`. So the compiler will never treat these types as
equal.

This raises a question: why are such types even useful?

Certainly, in positions which ascribe a type to a value, such as method or trait parameters or class fields,
they are not useful, since no instances exist which could be used in those positions.

But these are not the only places we use types: they are also widely used ephemerally during typechecking as
type parameters to generic types, and as bounds on other types which can serve as constraints on other types,
and can drive contextual search.

While these are more advanced uses of types, it is useful to be aware that they exist.

## Subtyping

A type, `A`, is a subtype of another type, `B`, if and only if `A` has all of the properties of `B`. Since the
type `Movable & Doubleclickable` has the properties of both `Movable` and `Doubleclickable`, it is a subtype of
`Movable` and it is a subtype of `Doubleclickable`.

We can partition the instances of `Movable` into those which are also instances of `Doubleclickable`, and those
which are not; and the instances of `Doubleclickable` may be partitioned into those which are also instances of
`Movable` and those which are not.

It may seem obvious, but whichever of these two approaches we take to defining the set of instances that are
both `Doublclickable`s and `Movable`s, the resultant set of instances will be identical and indistinguishable,
and is some intersection of the set of all `Doubleclickable`s and the set of all `Movable`s. (And this is where
the name, "intersection type", arises!)

That means that `Movable & Doubleclickable` and `Doubleclickable & Movable` are _identical_ types, as they
define exactly the same set of properties and exactly the same set of instances.

(Earlier versions of Scala used the `with` keyword to construct _compound types_, whose behavior is very similar
to intersection types, but which had subtle differences under certain operations which were ignored under
others. Due to the problems this caused, these have been replaced with intersection types in Scala 3.)

## Simplification of Intersection Types

In general, an intersection of several types, such as `A & B & C` can be written in any order, and refers to an
identical and indistinguishable type. `B & A & C`, `C & B & A` and every other permutation of `A`, `B` and `C`
will be the same type under the `&` type combinator.

Furthermore, repetitions of types in an intersection may be ignored. The type `A & A` is identical to just `A`,
and, for a more complex example, `A & B & A & C & B` would be identical to `A & B & C`.

Whilst it's also possible to use paretheses in a type intersection, these are redundant, as the order of terms
is insignificant. `A & (B & C)` is the same as `(A & B) & C`, and both are just `A & B & C`.

These simplification rules are used by the compiler whenever one type is compared to another.

## Subsumption

A further simplification may be performed if there is a subtyping relationship between any of the types in an
intersection. Since the presence of one type in an intersection already implies the properties of all of its
supertypes, there is no need to include the supertype in the intersection.

So, given our definition of `Doubleclickable` above, which has `Clickable` as a supertype, the intersection
type `Doubleclickable & Clickable` could be simplified to `Doubleclickable` as this provides sufficient
information for us to infer the inclusion of the properties of `Clickable` too. We say that `Clickable` is
_subsumed_ into `Doubleclickable`, and the process is called _subsumption_.

Likewise, the type `Movable & Clickable & Doubleclickable` could be simplified to `Movable & Doubleclickable`.
In general, any pair of types with a subtyping relationship between them appearing together in a type
intersection may be replaced by the more precise of the two types (i.e. the subtype), until every type in the
intersection is unrelated to every other type in the intersection.

## Duality

We saw at the beginning of this lesson how properties and instances are two alternative ways of defining types,
with a dual relationship between them.

This duality is further exemplified when we consider how an intersection of two types would be seen from each
perspective: most naturally, an intersection of two types is an intersection of their instances. But an
intersection type is a _union_ of their properties.

The intriguing dual relationship between instances and properties exactly reflects the dual relationship between
set-intersection and set-union. When we explore _union types_, in the next lesson in this topic, we shall take
this a step further as we see that they represent a _union of instances_ as well as an
_intersection of properties_.
