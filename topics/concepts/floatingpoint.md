## Integers

It's well known that computers work well with numbers. They can perform many millions of arithmetic operations
per second, without error.

But this fact is mostly attributed to operations on _whole numbers_; on integers. And in many ways, integers are
simpler to work with than other numbers. In Scala, this means the four integral types: `Long`, `Int`, `Short`
and `Byte`.

And the four arithmetic operations are defined for integers such that their results are always integers. For
addition, subtraction and multiplication, as long as the result of the operation is not too large, it can always
be represented precisely by another integer. The same is also true of division, but requires that it is defined
as [Euclidean division](https://en.wikipedia.org/wiki/Euclidean_division), where the given result of the `/`
operator is the _quotient_ of the division, while the `%` operator exists to provide the _remainder_.

We say that the integers are _closed_ under these operations, assuming that we do not produce numbers which are
too large to be represented.

## Real numbers

But integers work with only with discrete quantities: they are ideal for counting, but less good for measuring
continuous quantities, like lengths, weights, magnetic forces or the luminescence of a star, and these
quantities also occur very frequently in the physical world.

Most applied mathematics uses _real numbers_ to represent these quantities.

Both integers and real numbers are infinite. For integers, that means that they can be arbitrarily large—for any
integer we specify, we can always specify a larger one by adding one—but these larger and larger numbers are
only necessary when we are already working with large numbers.

In practice, the 32 bits used to represent an `Int`, and certainly the 64 bits which represent a `Long` are more
than sufficient for most needs, and we do not need to be concerned that we will have numbers so large that we
reach the limits of these representations.

For real numbers, not only can they be arbitrarily large, but they can be arbitrarily precise. For any two
real numbers, no matter how close they are to each other, it's always possible to find another real number that
fits between them. In fact, it's always possible to find infinitely many real numbers that fit between them!

So, unlike the integers, with the real numbers, we don't even have to be working with very large numbers to be
exposed to the need for more and more distinct numbers. And this presents a challenge for computers which can
fundamentally only operate on quantized data.

## Floating-point numbers

Floating-point numbers are an approach that is commonly used to mitigate computers' inability to work with real
numbers, and they provide representations for a wide range of real numbers.

Two floating-point types are natively supported on the Java Virtual Machine, `Float` and `Double`, which use
32 and 64 bits, respectively, to represent different numbers. That is to say, a `Float` can represent 2³²
different real numbers, and a `Double` can represent 2⁶⁴ different real numbers—or thereabouts. (There are a few
instances which do not represent valid numbers.)

Every number represented by a `Float` or a `Double` is a valid real number, but there are _infinite_ real
numbers, which means that only a subset of the real numbers can be represented exactly as a floating-point
number. The others must be approximated to the closest representable number.

That many of the floating-point numbers we work with are approximations is mitigated by the fact that the
approximations are nevertheless very close approximations in general, and will be correct to 6 or 7 decimal
places for `Float`s, and to 15 or 16 decimal places for `Double`s.

So the number π, which is 3.141592653589793238462643383... (with arbitrarily many more digits, if we ever needed
them) can be represented no more precisely than 3.141592653589793 as a `Double`. So our representation contains
an error of approximately 0.000000000000000238462643383. So while we do not have an exact representation, the
error is nevertheless tiny, and should be negligible for almost any purpose.

The floating-point numbers are also not evenly distributed across the range. Smaller numbers—those closer to
zero—are packed more densely than larger numbers (whether they are positive or negative). This is very practical
when applied to the physical quantities these numbers typically represent: a metre is hugely significant if we
are working at the scale of a human hair thickness, while it is insignificant if we are working with the
distances between stars.

The rounding error that occurs when a real number is converted to its closest floating-point representation will
be small, and will be roughly in proportion to the scale of the number. That does mean, however, that in
absolute terms, rounding errors can vary hugely between very small numbers and very large numbers.

## Representation

This should seem familiar to anyone familiar with scientific notation, which usually normalizes numbers to
representations written as a positive or negative number, less than 10.0 but not less than 1.0, multiplied by
10 raised to a power. For example, the following common scientific quantities can be written this way:

- Planck's Constant, 6.62607004 × 10⁻³⁴ m² kg s⁻¹
- Boltzmann Constant, 1.38064852 × 10⁻²³ m² kg s⁻² K⁻¹
- Avogadro's Number, 6.02214086 × 10²³ mol⁻¹

The number to which 10 is raised in each of these quantities can be thought of as the number of places the
decimal point has to be moved to the right (for positive numbers) or left (for negative numbers), inserting
additional `0`s as necessary, in order to write the number's full expansion.

For example, Avogadro's Number may be written as 602214086000000000000000, though it should be obvious that this
is less convenient than the form written in scientific notation.

Floating-point numbers have a similar representation on a computer. Of the 32 or 64 bits available to represent
numbers in `Float`s and `Double`s, most are dedicated to representing the normalized numerical detail of the
quantity, the _mantissa_, while some are responsible for indicating its scale, its _exponent_. Additionally, a
single bit determines whether the quantity is positive or negative, the _sign bit_.

While scientific notation conventionally uses decimals (that is, the digits `0` to `9`) multiplied by 10 raised
to a power, floating-point numbers instead use binary for their internal representation, so their form is,
_m_ × 2ⁿ, where _m_ is the mantissa and _n_ is the exponent.

The following table shows how `Float`s and `Double`s compare in their usage of bits:

Type     | Mantissa bits | Exponent bits | Sign bit | Total bits |
---------|---------------|---------------|----------|------------|
`Float`  | 23            | 8             | 1        | 32         |
`Double` | 52            | 11            | 1        | 64         |

And this results in the capacity to represent different ranges of numbers, as follows:

Type     | Largest representable number | Smallest representable number |
---------|------------------------------|-------------------------------|
`Float`  | 3.40282347 × 10³⁸            | 1.40239846 × 10⁻⁴⁵            |
`Double` | 1.7976931348623157 × 10³⁰⁸   | 4.9406564584124654 × 10⁻³²⁴   |

Both `Float`s and `Double`s offer a range of representable numbers that should be sufficient for almost any
purpose, and while we can have quantities, such as the estimate of the total number of atoms in the known
universe, which cannot be represented by a `Float`, a `Double` is more than sufficient to represent the number
of relationships between every atom in the known universe with every other atom in the known universe, which is
mindbogglingly large!

## Infinities and More

Sometimes if we produce `Float`s or `Double`s that are too large to be represented, either as positive or
negative numbers, the resultant values will be a special bit pattern which represents positive or negative
infinity.

Infinity is printed simply as `Infinity` or `-Infinity` when converted to a `String`. These can also be
generated by deviding any finite non-zero number by zero.

An infinite value is not an error, and it represents a valid state for a floating-point number. But there are
limits to its usefulness. It's possible to negate an infinite value, which changes `Infinity` to `-Infinity`,
but other operations generally leave it unchanged.

This can have some unintuitive consequences, for example, when calculating `Double.MaxValue*2/2`: attempting to
double the largest possible `Double` results in `Infinity`, but subsequently dividing by two—which we might
naïvely expect to restore the original `Double.MaxValue` value—returns `Infinity`.

Other operations, such as `0.0/0.0`, which are typically undefined will produce a result called `NaN`, which is
short for, "Not a Number". Like `Infinity`, `NaN` is not an error, but it usually represents the end of any
useful result from a calculation, and all subsequent operations involving a `NaN` will yield another `NaN`.

Unlike the value `Infinity`, which is considered equal to another `Infinity` value produced elsewhere, `NaN` is
unique amongst floating-point numbers in that it is not equal to any other value, including to itself! So the
predicate, `0.0/0.0 == 0.0/0.0` is always false, even though both sides are `NaN`.

This is commonly used to test whether a floating-point number is `NaN`, though the methods `Double#isNaN` and
`Float#isNaN` also exist and may be more instructive.

Finally, a quirk in the representation of `Double`s and `Float`s means that the sign bit may be `1` or `0` for
a number whose mantissa digits are all `0`, and this provides two distinct representations of zero.

Rarely, the distinction can be used to indicate something about the prior calculation. If, for example, we take
the smallest positive value representable as a `Double`, `Double.MinPositiveValue`, then negate it and divide it
by `2` (or more), the result will be `-0.0` rather than `0.0`. And dividing a finite positive number by `-0.0`
will yield `-Infinity` instead of `Infinity`.

These have slightly special behavior: while they have different representations, they are considered equal, but
are printed as `0.0` and `-0.0`.

## Caution

Despite floating-point numbers providing a good set of compromises for representing different quantities within
a large range, with a good degree of precision, when we work with them we should not forget that they are not
the same as real numbers, and rounding errors are possible with many calculations.

Furthermore, these errors may accumulate after many operations to produce more significant deviations from the
correct answer, often in ways that are difficult to predict.

We should therefore have realistic expectations about the accuracy of results we get from operations involving
floating-point operations, and formulæ which are algebraically equivalent may produce different results when
applied to the same floating-point numbers, due to differences in the errors in representing intermediate
results.

Very careful analysis can help to minimise such errors, but it is nevertheless complex, and beyond the scope of
this course. So we are best advised to expect some amount of inaccuracy in results represented by `Double`s and
`Float`s, and to use them cautiously.

The [IEEE 754 Standard](https://en.wikipedia.org/wiki/IEEE_754) provides a full specification of the
implementation of floating-point arithmetic on the JVM.
