## Strings as Binary

When we work with strings in Scala, we almost never have to worry about how the characters which make up the
string are stored in memory: Scala gives us a `Char` type for representing different characters in `String`
instances, and the type can represent all _Unicode_ characters.

But all the textual data we work with in a Scala program must ultimately be stored in memory _somehow_ and
processed as binary data (which we can think of as an `Array[Byte]`). So while the `Char` abstraction hides this
detail from us, some translation between characters and their binary representations is nevertheless happening
whenever strings are placed into or retrieved from memory.

This abstraction works seamlessly _within_ any Scala program, because strings of text are always represented
internally in exactly the same binary format, and are presented to us consistently through the `String` and
`Char` types.

When we print a string of text from Scala, it will need to be passed to the underlying operating system, which
may involve converting it to a format the operating system understands. But that is also handled seamlessly.

The process of translating from strings of characters to binary data is called _encoding_, while the reverse
process from binary data to strings of characters is called _decoding_. The algorithm that defines the mapping
is called a _character encoding_.

## Information Interchange

When we need to exchange character data with other systems, sending or receiving it across the network or by
storing it on disk, we need to consider how the data can be exchanged in such a way that the other system is
able to read it correctly. And unfortunately, different answers will apply in different scenarios.

For many tasks that involve information interchange, we will use a third-party library which provides us with
a high-level interface, defined in terms of `String` (and maybe `Char`) types, for exchanging character data,
and there is no need to consider what encoding is used. Assuming we trust the library to do the right thing
(which should not necessarily be taken for granted!) this is ideal because it avoids the potentially-problematic
choice of character encoding.

Other libraries may choose a default character encoding for us, while offering an option to override that
default with a specific choice.

And some libraries may always require a character encoding to be explicitly specified, or may offer only
interfaces defined in terms of `Array[Byte]` rather than `String`, thereby forcing any encoding to be performed
beforehand.

All these varieties exist commonly, and it helps to be aware of when encoding or decoding is necessary, and
also when we should be explicit about specifying a character encoding, or when it's acceptable to accept a
default.

## Encoding and Decoding

In general, when exchanging character data between different systems, there will either be a common, invariant
and pre-established character encoding agreed between the source and the destination, or the encoding used will
be sent along with the character data.

This specification of the encoding may be sent within the character data itself in a machine-readable header
close to the start of the data (that the destination system expects to read), or in a "prelude" or "handshake"
performed before the data is sent.

Different protocols and formats, like HTTP or XML, use different methods for specifying the character encoding
in use, so how that is communicated between different systems must be learned separately for each protocol or
format.

## ASCII

While many dozens of different character encodings exist (each with variants, as the set of characters they
represent evolves over time), it is useful to know about three that are currently in most widespread usage
today: `ASCII`, `UTF-8` and, primarily in the English-speaking world, `ISO-8859-1`.

ASCII, or the _American Standard Code for Information Interchange_, defines 95 different alphabetic, numeric and
symbolic characters that can represent most English-language text, and much text written in other Latin-based
alphabets, such as French, German, Norwegian or Polish—with the compromise that many accented letters must be
replaced with their closest equivalent letters from the 26-letter English alphabet.

In addition to these 95 printable characters, ASCII encodes a further 33 non-printable characters, or "control
characters", that mostly have historical meaning, but also include characters representing newlines.

Together, these 128 characters can be stored in seven bits. A single 8-bit byte can therefore hold any ASCII
character (with a single bit unused) and this is how ASCII text is normally transmitted, though forms of
compression do exist which avoid wasting one bit for every character, and can store ASCII in ⅞ of the space it
would normally take.

But this means that there is not a one-to-one mapping between bytes and ASCII characters, and different software
may respond in different ways when asked to convert a byte to an ASCII character, if that byte is not within the
7-bit range that ASCII defines: it may skip the character, replace it with another character such as `?`, ignore
the problematic bit, or fail completely.

## ISO-8859 encodings

`ISO-8859-1` (which is sometimes known as `Latin-1`, and closely related to the `Windows-1252` encoding) is an
8-bit encoding, meaning that every byte translates to a valid printable or non-printable character and every
character that is possible to encode will use exactly one byte. It is the most widely-used such encoding on the
Internet.

The first 128 characters of `ISO-8859-1` are the same as ASCII's, and the remaining 128 characters provide
additional characters which, when combined with the English letters defined in ASCII, support about thirty
different recognised world languages which generally use the Latin alphabet. These are mostly the languages used
in western Europe.

In fact, a series of less widely-used character encodings exist, numbered `ISO-8859-n` for `n` up to `16` (with
a couple of omissions), each combining the ASCII character encoding with additional characters to support other
languages.

Sharing the characters defined in the first seven bits of these encodings with ASCII means that text encoded as
bytes with ASCII is already valid `ISO-8859-1` (and indeed valid `ISO-8859-13`, for example, too).

The direct correspondence between bytes and characters for `ISO-8859-1` also means that a text file with this
encoding which is _n_ bytes long, will also contain _n_ characters.

## Unicode

The ASCII standard was defined in the 1960s, while the `ISO-8859-n` character encodings were mostly defined in
the 1980s. Since then, the _Unicode_ standard has evolved to define a mapping from integers to the characters of
almost all known written languages, not to mention numerous other symbols used in various forms of writing.

These are called _codepoints_, and the Unicode standard currently specifies over 143000 of them.

Obviously, this is too many to fit into a single byte, or even two bytes. But it's possible to encode the most
commonly-used characters with a single byte, and some of the less common characters with two or more bytes.

And this is what the `UTF-8` encoding does. Each Unicode codepoint can be encoded as one, two, three or four
bytes. Like the `ISO-8859` encodings, the first 128 codepoints of Unicode are identical to ASCII, meaning that
ASCII-encoded text is also valid `UTF-8` text.

## Conversions

A `String` in Scala can be encoded in a binary format only with a character encoding. Unfortunately, the Java
standard library will use the current system encoding by default, which means that the same program may produce
different results when run on different machines, if those machines have different character encodings. This is
undesirable if we require portability of our software.

Computer operating systems usually define a system-wide character encoding which provides a default
interpretation for text files, and other fundamental aspects of the computer, such as filenames.

Since most, if not all, critically-important strings used on a computer will use only ASCII characters, the
binary representation of these strings will be identical whether the system encoding is set to `UTF-8` or
`ISO-8859-1`, and none have the opportunity be misinterpreted.

For many years, that worked adequately for computers which mostly consumed only text which they had themselves
produced, but since the Internet was created, it became much more common for text to be exchanged between
computers with different system encodings.

It is therefore usually preferable that we be explicit about the character encoding whenever a conversion is
necessary.

We can obtain the bytes representing an encoded string using the `String#getBytes` method. `"Hello".getBytes`
will always return the byte array, `Array(72, 101, 108, 108, 111)`, because the string `"Hello"` uses only ASCII
characters, which encode to the same bytes whatever commonly-used system encoding is in use.

But `"Café".getBytes` may return, `Array(67, 97, 102, -61, -87)` on a computer using the `UTF-8` encoding, or
`Array(67, 97, 102, -23)` on a computer using `ISO-8859-1`, since the letter `é` is encoded to different byte
sequences (and also different numbers of bytes) in each character encoding.

Since the 7-bit subset of the byte range used for ASCII corresponds to non-negative numbers, a character which
is encoded to a negative number cannot be an ASCII character.

Furthermore, the definition of the `UTF-8` encoding of a Unicode character is such that every character which
requires two or more bytes will _never_ be encoded as bytes which are valid ASCII characters, which means that
every byte in a multi-byte sequence will be a negative number.

(This interpretation assumes Scala's interpretation of bytes as _signed integers_, though the raw bytes are just
eight bits of memory, without an inherent numerical range, and could be interpreted differently in other
scenarios.)

We can be more explicit about the encoding by specifying the character encoding as a string itself, in a
parameter to the `String#getBytes` method, for example by calling, `"Café".getBytes("UTF-8")`.

We can also examine the result of trying to encode the string `"Café"` using ASCII: `"Café".getBytes("ASCII")`
will return `Array(67, 97, 102, 63)`.

This may seem surprising, since the letter `é`, like many other accented letters which appear only rarely in
English words, cannot be encoded with ASCII! But we will inspect what has happened later, when decoding this
byte array...

In general, we can decode an `Array[Byte]` to produce a `String` using the one- or two-parameter constructor for
`String`. The first parameter is always the array containing the bytes which represent the string, while the
second optional parameter is a `String` representing the encoding to be used.

For example, `String(Array[Byte](72, 101, 108, 108, 111))` will produce the string `"Hello"`, interpreting the
five given bytes using the system encoding (which we know will work for this particular example, since every
byte is in the common ASCII subset).

But for an array of bytes originating elsewhere, called `bytes` for example, we should specify the character
encoding explicitly, for example with, `String(bytes, "UTF-8")` in case our code is executed on a system whose
operating system uses a character encoding other than `UTF-8`.

## Missing Characters

If we go back to the example of encoding `"Café"` using ASCII, we earlier obtained the byte array
`Array(67, 97, 102, 63)`. In theory, we should be able to decode those bytes to get back the original string
value. But `String(Array[Byte](67, 97, 102, 63), "ASCII")` will produce a slightly different string: `"Caf?"`.

That's because the `é` character was indeed impossible to encode in ASCII, but Java nevertheless made a
compromise and used a substitute character, `?`, in place of `é`.

Sometimes it would be preferable to know that it was not possible te encode a character precisely, but the
`getBytes` method always returns a result and will silently replace any characters it cannot encode with
characters it can. We shouldn't forget that this may happen if our destination encoding cannot represent the
full range of Unicode characters that the `String` type may contain.

## Mismatched Encodings

Textual data can be transmitted quite freely across the Internet, and there are a variety of different ways in
which the encoding of that data may be specified.

For example, an HTML web page served over HTTP may have,
- an HTTP `Content-Type` header specifying the character encoding,
- an HTML `<meta>` tag specifying the equivalent of that HTTP header as HTML,
- a `<meta>` tag specifying the character encoding,
- an XML processing instruction to use a particular character encoding,
- a distinctive _byte-order mark_ (BOM) character at the start of the text indicating that it is `UTF-8`, or,
- none of these, leaving the interpretation of the textual content to defaults.

As well as providing many different ways to specify how the content is encoded, there are also many ways to
specify the character encoding _incorrectly_! And any inconsistency between different ways of specifying the
encoding could produce inconsistent results on different system.

Consequently, we frequently see examples of mismatched encodings, where binary data representing text encoded
with one character encoding is decoded using a different character encoding.

The most common mismatch involves both `UTF-8` and `ISO-8859-1`, and we can simulate the consequences of such a
mismatch with a couple of examples which encode a string with one encoding, then attempt to decode it with
another.

1. `String("Café".getBytes("UTF-8"), "ISO-8859-1")` produces the string, `"CafÄ©"`, and
2. `String("Café".getBytes("ISO-8859-1"), "UTF-8")` produces a string, `"Caf "`, containing an unprintable
   character at the end which looks like a space.

Since these incorrect encodings both appear frequently, it's useful to learn to identify them, as this can be
the first step towards identifying the source of the problem.

Of course, the spurious characters in the example above, `Ä©`, or equivalently the missing characters, are not
the same each time, so the best indication of the nature of the mismatch is the number of characters that
appear in the decoded text, since `ISO-8859-1` always assumes a single byte per character, while `UTF-8` may use
more than one byte for one character.

So if the text appears to contain two or more unexpected characters where a single non-ASCII character would be
more likely, then this is likely to represent `UTF-8`-encoded text being interpreted as `ISO-8859-1`, while
missing characters, or fewer characters than expected being displayed probably represents the reverse.

## The Future

The JVM uses Unicode internally to represent strings, and within the bounds of a Scala program, we never need to
consider how `String` instances are stored in memory. But further afield, text encoded with different character
encodings (often created with software that predates Unicode) is still widespread, despite the continued
adoption of the Unicode standard across operating systems, editors, programming languages and elsewhere.

Every modern operating system these days uses Unicode by default, so while Unicode may one day become the only
encoding in use, its predecessors are likely to have a long legacy.

Therefore, as long as that remains the case, we need to be aware that text may be stored using different
character encodings, and cognizant of the challenges of working in a multi-encoding world.
