We saw an example earlier of a pattern using two `::` extractors, `case head :: head2 :: Nil`. The compiler
must make a decision about what order to interpret this. Should the first or the second `::` be applied first?
1. `case (head :: head2) :: Nil`, or
2. `case head :: (head2 :: Nil)`

Remembering that `head` and `head2` are both element types, and that `::` always prepends an _element_ to a
_list_, we can see that the second case is the only interpretation that makes sense: `(head :: head2)` would
not typecheck unless `head2` were a _list_.

But the choice of precedence is determined not by type, but by the _name_ of the extractor. Specifically, any
infix extractor whose final character is a colon, such as `+:`, will be parsed _right_associatively_. This is
similar to the behavior of infix symbolic methods which end in a `:` being _right-associative_, but for
extractors—objects which must be accessible in the scope they are used, without a prefix, not members of one
of the operands—it means that in a series of applications of the same extractor, it will be as if the operands
are grouped from the right. So,
```scala
case a +: b +: c +: d +: e =>
```
would be equivalent to,
```scala
case a +: (b +: (c +: (d +: e))) =>
```
if we were to include parentheses explicitly. A left-associative extractor, such as `:+` would group from the
left, like so:
```scala
case (((a :+ b) :+ c) :+ d) +: e =>
```

The associativity of the extractors determines the structure of the pattern, but because patterns are evaluated
in an "inside-out" order, the _evaluation order_ is reversed from what we might expect. In,
```scala
case a :: (b :: c) =>
```
the first extractor to be applied at runtime will be the `::` between `a` and `(b :: c)`, followed by the `::`
between `b` and `c`. So the _evaluation order_ will be left-to-right for right-associative extractors, and
right-to-left for left-associative extractors, but only because we are working with a _pattern_ and not an
_expression_.

## Precedence (intermediate/advanced)

Combinations of infix extractors are possible in patterns, to provide expression-like syntax. Consider the
following definitions for a simple expression language:
```scala
sealed trait Expr:
  def *(expr: Expr): Expr = Intersection(this, expr)
  def +(expr: Expr): Expr = Union(this, expr)

case class Union(left: Expr, right: Expr) extends Expr
case class Intersection(left: Expr, right: Expr) extends Expr
case class Ref(name: String) extends Expr
```

These definitions would permit us to construct an expression such as,
```scala
a * b + c * d
```
where `a`, `b`, `c` and `d` are all expressions.

The additional definitions of extractors for these types,
```scala
object * :
   def unapply(expr: Expr): Option[(Expr, Expr)] = expr match
      case Intersection(a, b) => Some((a, b))
      case _                  => None

object + :
   def unapply(expr: Expr): Option[(Expr, Expr)] = expr match
      case Union(a, b) => Some((a, b))
      case _           => None
```
would permit us to match on such an expression, too. For example,
```scala
case a * b + c * d =>
```

But both the construction of the expression and the deconstruction rely on the precedence order of the symbolic
operators, `*` and `+`. We require that both the expression and the pattern should be interpreted as,
```scala
(a * b) + (c * d)
```
because we want `*` to have higher precedence than `+`. Thankfully, this is exactly how Scala will interpret
them, and the same precedence order that applies to term operators also applies in patterns, based on the first
character of the operator. Here is a reminder of that order, from highest to lowest precedence:
- `*`, `/`, `%`
- `+`, `-`
- `:`
- `=`, `!`
- `<`, `>`
- `&`
- `^`
- `|`
- alphabetic characters

## Ambiguous cases

Scala can parse patterns of infix-operators unambiguously, except in cases where left- and right-associative
operators of the same precedence are mixed. That is not a common occurrence, but it's useful to remember that
a combination of extractors in a pattern such as,
```scala
case a :: b :+ c =>
```
contains one which is right-associative (`::`), one which is left-associative (`:+`) while both have the same
precedence (`::` and `:+` both begin with a `:`). The compiler would not be able to make a choice between
interpreting this pattern as,
```scala
case (a :: b) :+ c =>
```
and,
```scala
case a :: (b :+ c) =>
```
so it considers it a compile error.
