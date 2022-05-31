Computers were made for working with numbers, so operations on numbers (or, arithmetic) are a core part of
programming in Scala, as with most other programming languages.

When we work with number types in Scala—that is, `Double`s, `Float`s, `Long`s, `Int`s, `Short`s or `Byte`s—we
can use the symbolic arithmetic operators to combine them in expressions, such as the following:
- `4 + 2`
- `4.17/18.0`
- `1 - 2*6.909946/8 + 0.7274865`
- `x/100.0`
- `a - b*c`

In these expressions, the symbols `+`, `-`, `*` and `/` correspond to addition, subtraction, multiplication and
division, respectively. The `×` symbol is not used as it is too easily confused with the letter `x`, and is not
easy to type on a standard keyboard. The symbol `÷` is not used for similar reasons.

But these operators are nothing more than methods defined on each of the number types, all of them taking a
single parameter, while written in _infix_ style, that is, placed between the two operands without a `.` or
parentheses.

In Scala, term names do not need to be alphanumeric, so it is possible to define methods with symbolic names
like `/` or `#-#` as long as these do not conflict with existing built-in operators, like `#` or `<-`. It is
not possible to mix symbolic and alphanumeric characters in a term name except when there is an underscore
between the alphanumeric characters and the symbolic characters in the name, for example, `value_+`. This is
rarely useful, though.

So an arithmetic expression such as `4 + 2` could be written as `4.+(2)` (which is valid Scala syntax, even
though we would almost never use it), as it is nothing more than the `+` method defined on the value `4` which
has the type `Int` and which takes a single parameter, `2`, also with the type `Int`.

Thinking about arithmetic expressions in this way means that we can reason about them like any other method
application. The fact that we can write them with clean syntax that mirrors a mathematical expression is
convenient, but only superficial.

## Precedence

However, when we combine addition, subtraction, multiplication and division together in the same expression, it
is important to know what order the operations are applied in, as it can affect the result.

For example, it is plausible—though highly unconventional—that the mathematical expression, _1 + 3 × 2_, could
be interpreted in such a way as to result in _8_, by performing the addition operation before the
multiplication. Naïvely, this would seem logical in a strict left-to-right evaluation order. Nonetheless,
by convention, we should first multiply _3_ by _2_ and _then_ add the result to _1_, to get _7_.

Similarly in Scala, a `*` operation will be evaluated before a `+` operator. We say that `*` has
_higher precedence_ than `+` (or conversely, that `+` has _lower precedence_ than `*`).

Expressions surrounded by parentheses, `(` and `)`, will always be evaluated before expressions which are not,
so `(1 + 3) * 2` will simplify first to `(4) * 2`, and hence evaluates to `8`, while the parameterless variant,
`1 + 3 * 2` will be `7`.

In general, Scala's precedence order dictates the order of evaluation for a series of three or more terms
combined with two or more different infix operators. The following table specifies the fixed order of precedence
that is built into the Scala language.

| Symbols                 | Precedence |
|-------------------------|------------|
| all other symbols       | high       |
| `*`, `/`, `%`           |            |
| `+`, `-`                |            |
| `:`                     |            |
| `<`, `>`                |            |
| `=`, `!`                |            |
| `&`                     |            |
| `^`                     |            |
| `\|`                    |            |
| alphanumeric characters | low        |

An infix operator (whether symbolic or alphanumeric) which starts with a character closer to the top of this
table will be applied to its operands before one appearing closer to the bottom of the table. We can see that
`*` appears in the row immediately above `+`, which gives us the precedence for multiplication (and division)
over addition (and subtraction) that we need for arithmetic.

If adjacent terms in an expression are combined with operators with the same precedence, then they are evaluated
in left-to-right order. This has no bearing on arithmetic involving only the `+` or `*` operators, as they are
defined to be symmetrical—`a + b` always equals `b + a`, and `a*b` always equals `b*a`—but it becomes
significant for subtraction or division. For example, `a - b - c` will be interpreted as `(a - b) - c`, which
is not the same as `a - (b - c)`, at least for numerical values.

Operators starting with other characters in this order are commonly used in different contexts. For example,
several operators on collections start with `:`, while operations on `Boolean`s use `&`, `|` and `^` and
require, by convention, that `&` has higher precedence than `|`.

It is possible to define other operators like `∍` or `∾`, which would automatically take the highest precedence,
but this is generally discouraged in most cases as these characters are awkward to type.

Symbolic operators of any sort are often discouraged because they are harder to remember than alphanumeric
method names. But while alphanumeric methods can be also be used in the infix style, they will all take the same
precedence (which also happens to be lower precedence than any symbolic operator).

An operator's precedence is determined only by the position of the first character of its name in this list,
and can only be changed by renaming or creating an alias for that operator.

While it's not possible to customize the precedence order of user-defined methods, a fixed order of precedence
has the advantage that it's more straightforward to understand the meaning of code without knowing the exact
definition of every operator.

Nevertheless, to make it easier to read expressions involving mixed precedence, whitespace can be used to make
higher-precedence operators appear closer together than lower-precedence operators.

For example, we can write `x*x - y*y` instead of `x * x - y * y` to make it visually clearer that `x*x` and
`y*y` should be evaluated first. Adding or removing whitespace has absolutely no effect on the meaning of the
code; it's just to make the code more readable by other programmers.

## Associativity

Usually, an operator will _bind_ to the left operand, and _applies_ to the right operand. That is to say, the
expression `a / b` is semantically identical to `a./(b)`, and is as true for `/` as it is for most other
operators.

The exceptions to that rule are symbolic operators whose final character is a colon, `:`. So the expression
`a %: b` is semantically identical to `b.%:(a)`: the left and right operands appear to switch places, though
more importantly, `%:` is a member of the term written to its right, not the term to its left. These operators
are called _right-associative_, while the majority are _left-associative_.

Sometimes this leads to a parsing ambiguity, when an operand stands between a right-associative and a
left-associative operator, for example:

```scala
val y = 1 :: x :+ 2
```

Should this be interpreted as, `(x :+ 2).::(1)`, or `(1 :: x).:+(2)`? The ambiguity occurs only because both
`:+` and `::` both _start_ with the same character, `:`, while one terminates with `+` and the other terminates
with `:`. In any case, Scala considers the code to be unclear, so it will report an error.

## Number Types

Arithmetic operators can be used between any two numeric types. The return type from an addition, subtraction,
multiplication or division between values of any two numeric types will always be an `Int`, `Long`, `Float` or
`Double`, and the return type will be an `Int`, except when either or both of the operands has a type of
`Long`, `Float` or `Double`. In these cases, the latter of the two types, in the order given here, will be
the return type.

For example,
- multiplying a `Long` and a `Float` will return a `Float`
- dividing a `Byte` by a `Double` will return a `Double`
- adding a `Short` to a `Long` will return a `Long`
- subtracting an `Byte` from a `Byte` will return an `Int`

## Constant Folding

The Scala compiler can sometimes perform some simple arithmetic on constant values at compiletime to save that
computation needing to be done at runtime.

For example, an assignment such as,
```scala
val x = 2*3*7 + 5*7*11
```
would produce Java bytecode which assigns the value `427` to the field indicated by `x`, and the numbers `2`,
`3`, `7`, `5` and `11` would not be apparent in the compiled bytecode for this statement.

Of course, this same calculation could have been simplified by us too; the compiler has no more information
available to it than we have. Very occasionally, though, it may be more instructive to anyone reading our code
to write a number as an arithmetic expression instead of its simplified form.

For example, here,
```scala
val oneWeek: Int = 60*60*24*7
```
it is much easier to see _why_ `oneWeek` is defined as `604800` when we write that number as `60*60*24*7` than
had we written it as a simple literal. The compiled bytecode would be identical in either case.
