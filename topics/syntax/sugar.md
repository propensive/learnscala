## Sweetening Syntax

All programming languages are, for the most part, regular in their use of syntax: the same syntactic patterns
are used over and over again, and the regularity and consistency make code easy to interpret.

But occasionally that regularity can make certain constructions more cumbersome, verbose or less intuitive to
read and write. Or sometimes, the introduction of new syntax—something which has no valid interpretation
otherwise—can help to make certain constructions within in code clearer.

We typically refer to the introduction of syntax rules of this nature as _syntactic sugar_; a small addition to
the routine syntax to make it sweeter (or just easier to consume)!

The classification of syntax rules as syntactic sugar is often open to interpretation. But as a minimum, to call
a rule "syntactic sugar", it should provide an alternative way of writing some syntax construction that is
already possible using more widely-used syntax, and to translate between these alternatives should only rely on
local reasoning; no information from any other part of the program should be required to switch from one syntax
to the other.

This lesson will look at several different kinds of syntact sugar that are available to make Scala's syntax a
little "sweeter".

The word _desugar_ is also commonly used to describe the translation from "sweetened" to "routine" syntax,
suggesting the "removal of syntactic sugar".

## Update methods

We have already seen that a method with the special name, `apply`, may be elided at the call-site, so any call
of the form `object.apply(value)` can be rewritten more concisely as `object(value)`.

In addition to the special method name, `apply`, which is typically used for _retrieving_ values, Scala provides
`update` methods for _providing_ values, and these undergo a similar transformation at the call-site.

A typical `update` method will take two parameters, where the first represents some "key" within the object to
be updated, and the second provides a new "value" for that key. For example, defining,
```scala
class Relations():
  private var refs: List[String] = Nil
  
  def update(source: String, destination: String): Unit =
    refs = s"$source -> $destination" :: refs
```
would allow a `Relations` instance, `r`, to be modified by calling,
```scala
r.update("Beta", "Delta")
```
but using syntactic sugar at the call-site, this can be simplified to:
```scala
r("Beta") = "Delta"
```

This can appear to be more intuitively updating the `"Beta"` key to the value `"Delta"`, while in our example,
the actual modification taking place inside the `Relations` object just prepends a descriptive string to a
list—there are no constraints on the exact nature of an `update` method's implementation.

More generally, an `update` method may take as few as a single parameter, or as many as we like, and while the
last parameter will always correspond to the right-hand side of the assignment, any parameters which come before
it will correspond to the key, for example,
```scala
Deck(Jack, Spades) = true
```
would desugar to the invocation,
```scala
Deck.update(Jack, Spades, true)
```
and would require a definition such as,
```scala
object Deck:
  def update(rank: Rank, suit: Suit, visible: Boolean) = ???
```

But likewise, a single-parameter `update` method could be invoked with,
```scala
Field() = 8
```
which would desugar to,
```
Field.update(8)
```

The transformation is carried out somewhat naïvely by the compiler, so details such as types and overloading are
not taken into account when desugaring an apparent assignment (only "apparent", as the left-hand side of a real
assignment must be a simple identifier, and not a term with parentheses) into an `update` invocation.

But this naïvity brings flexibility: it means that an `update` method may be defined in any way for which the
desugared code will compile. This permits other Scala features such as `using` parameter blocks, inferred type
parameters, or repeated arguments to be included in the definition. By virtue of this same flexibility, an
`update` method can also return a value, even though `Unit` is a more typical return type.

Scala's mutable `Map` collections are the most common example of this syntax.

## Setters

A common design pattern in Java is to use "getters" and "setters" to access members from a class or object.
These are methods—called `getX` which returns a value, often with a corresponding `setX` that takes a value of
the same type as a parameter—that can retrieve or update a mutable variable.

But since they are _methods_, in addition to retrieving or updating a variable, they can perform other actions
at the same time, which isn't possible when accessing that variable directly.

Usually calling a getter or setter would not perform any complex or slow operations, since their primary intent
is to _get_ and _set_ respectively, which are meant to be trivial operations. But a getter could, for example,
log the number of times it is called, and a setter could perform some transformation on its input value before
storing it, or could update a cache.

A getter is really nothing more than a method, and in Scala there's no convention to name a getter method with
the prefix `get`. So a getter method called that would be traditionally called `getValue` in Java would just be
called `value` in Scala.

However a setter method called `setValue` in Java would be named `value_=`. Although this may look like an odd
name, it is just that: a name. The `_=` at the end is part of the identifier, and relies on Scala's ability to
combine alphanumeric and symbolic characters in the same identifier _only_ when there is an underscore between
them.

Here's how the getter and setter might look in an object definition:
```scala
object Cell:
  private var intValue: Int = 0
  
  def value: Int =
    println("Accessed value")
    intValue

  def value_=(v: Int): Unit =
    println(s"Set new value to $v")
    intValue = v
 ```   

This unusual choice of name will be recognized by the compiler to enable the syntactic sugar which allows it to
be invoked using assignment syntax, so,
```scala
Cell.value = 4
```
will be interpreted as a call to:
```scala
Cell.value_=(4)
```

Note, however, that the syntactic sugar is enabled _only_ if both a getter method with a name corresponding to
the setter is defined on the same object: without an appropriately-named getter definition for `value` on
`Cell`, the statement `Cell.value = 4` would _not_ desugar.

This allows code which looks like a simple assignment to perform more interesting computations. Although the
example above is defined on a singleton object, there is nothing to restrict a setter being defined on a class
or trait method, and being available on instances of that template.

And setters may be imported into local scope and used directly, for example,
```scala
import Cell.*
value = 9
```

It is even possible for a method such as `value_=` to return a type other than `Unit`, so code which looks like
an assignment may even choose to return the newly-set value, or the object containing it, or something entirely
unrelated.

While this facility remains possible, we should, of course, try to make our code intuitive for others to use, so
returning a value other than `()` might not be desirable!

Scala's setter syntax provides similar functionality to its `update` methods. The most distinguishing difference
is the presence or absence of parentheses on the left-hand side of the `=` symbol, which determines
unambiguously how an assignment-like statement can desugar.

## Unary methods

It is common, in particular with arithmetic operations, to apply a unary (or, single-operand) operator to a
number, or other value, and these are commonly written as prefixes to that value, for example,
- `!x` to calculate the logical _NOT_ of a boolean value, `x`
- `-number` to negate the value, `number`
- `~value`, in a more domain-specific context, to indicate approximation of `value`

However, the methods we define normally come after (i.e. follow, when reading left-to-right) the value they are
called on, at the call-site, whereas these prefix operators _precede_ the value.

Scala works around this by desugaring any occurrence of one of four valid prefix operators—`!`, `~`, `+` and
`-`—into four method applications, applied to the value: `unary_!`, `unary_~`, `unary_+` and `unary_-`.

So, we could define a method, `unary_~` on a type,
```scala
case class Complex(real: Double, imaginary: Double):
  def unary_~ = Complex(real, -imaginary)
```
and use it as follows,
```scala
val x = Complex(2.8, -1.1)
val y = ~x
```
where Scala would transform the definition of `y` to,
```scala
val y = x.unary_~
```

Again, this transformation is performed naïvely, so a unary method definition can return any type, and may
be defined with type parameters or `using` parameters.

Only these four symbols can be used as prefix operators, and only single-character operators are permitted. It
is also impossible to use more than one prefix operator at a call site, so even if `~x` and `+x` are both valid
(because `unary_~` and `unary_+` are defined on `x`), the expression `~+x` is not valid: we would need to write,
`~(+x)`.

## Mutation Operators

When we work with mutable variables, it is common to want to update the variable to a new value by changing its
existing value in some way.

For example, we might want to "increment `count` by `1`" or "double `x`".

Normally, we would write these instructions as,
```scala
count = count + 1
```
and,
```scala
x = x*2
```

But there is syntactic sugar for performing these operations which avoids repeating the variable identifier
twice, provided the variable we are mutating appears as the _first operand_ of a _symbolic_ operator.

Instead of the examples above, we can write,
```scala
count += 1
```
and,
```scala
x *= 2
```

The transformation to the simplified syntax requires eliding the first operand completely, and moving the
symbolic operator to immediately precede the `=` assignment symbol, without any space.

It even works for symbolic operators more than one character long, but care should be taken with
right-associative operators (those which end in a colon, `:`) as their first operand is the _right operand_
rather than the _left operand_.

So, prepending an element to a `List[Int]` variable could be achieved with,
```scala
var xs: List[Int] = List(2, 3)
xs ::= 1
```
which would desugar to,
```scala
xs = xs.::(1)
```
which is the same as,
```scala
xs = 1 :: xs
```
when the `::` operator is written in more typical infix-style.

This syntactic sugar relies on one further criterion in order to be applicable: the value we are mutating must
_not_ already have a method with the same name as the mutation operator, otherwise that operator will be
prioritized over the syntactic sugar.

## Orthogonality

Scala's different examples of syntactic sugar work, wherever it's relevant, independently of each other. So they
can be combined in the same expression without any conflict.

In particular, mutation operators can be combined with both setters or `update` methods, using the corresponding
getter (which must exist to enable setter syntax at all) or `apply` method if it exists (remembering that it is
possible for the `update` method to exist without it), respectively.

So a statement such as,
```scala
x += 1
```
where `x` and `x_=` are getter and setter methods in scope, would desugar to,
```scala
x_=(x + 1)
```
while,
```scala
field("epsilon") -= value
```
would become,
```scala
field.update("epsilon", field.apply("epsilon") - value)
```
provided, of course, that there is no operator called `-=` defined directly on the return type of `field.apply`!

In our example, `"epsilon"` is a pure literal string, but whatever expressions are used for the keys to the
desugared `update` and `apply` methods will be used twice, but evaluated only once.

## Summary

These four syntactic enhancements to Scala provide us with more expressive syntax. It comes at a small cost that
we need to be aware of some additional source code transformations in order to fully understand how certain
statements and expressions are being interpreted.

But the transformations rely only on local reasoning, and by design they are all intuitive: they provide
flexibility to library designers to enable more usable APIs.
