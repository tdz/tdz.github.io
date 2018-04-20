---
layout:     post
title:      Safe Integer Conversion in C
date:       2018-04-20 15:30:00 +0200
tags:       [c, casting, conversion, integer, picotm]
og-image:   share2.png
---

Integer conversion is tricky and error prone. When converting from a larger
type to a smaller type, overflows and underflows can happen. For conversion
of signed values, the rules are often obscure and the compiler seems to
perform some *dark magic* in the background.

In this post, we'll looks at the situation in C and develop a general
function to cover all the corner cases. The type-conversion feature is
part of [picotm][picotm] 0.11 and later, which also covers integration
with larger software applications.

<!-- excerpt -->

There are two central problems in integer type conversion. The first are
conversions from types with wider range (i.e., more bits) to types with
smaller range (i.e., less bits). Here we have to be careful to not overflow
or underflow the result type.

The second problem is the conversion from signed values (i.e., negative
numbers) to unsigned types, which cannot represent signs. To compare signed
and unsigned types, the C compiler performs some *interpretation* of the
values, which can lead to surprising results.

#### Case 1: Unsigned to Unsigned

Let's start with something simple. Let's cast a value of type
`unsigned short` to `unsigned char`. In our example, the former type contains
16 bit, the latter contains 8 bit.

~~~ c
    unsigned short val_u16 = 1;
    unsigned char val_u8 = (unsigned char)val_u16;
~~~

With a value of *1,* this works well. Let's try with a different number.

~~~ c
    unsigned short val_u16 = 456;
    unsigned char val_u8 = (unsigned char)val_u16;
~~~

As `unsigned char` can only represent values ranging from 0 to 255,
the casting operation cannot give a correct result. Thus we test this case
and report an error accordingly.

~~~ c
    unsigned short val_u16 = 456;
    unsigned char val_u8;

    if (val_u16 > UCHAR_MAX) {
        report_error_and_abort(EDOM);
    }
    val_u8 = (unsigned char)val_u16;
~~~

Our test checks the value stored in `val_u16` against the constant
`UCHAR_MAX`, which represents the maximum value that a variable of
type `unsigned char` can store. If the test fails, we stop and report
a domain error with the errno code of `EDOM`.

The function `report_error_and_abort()` is provided by the application.
`EDOM` is defined in the C standard header `<errno.h>`, `UCHAR_MAX` is
defined in `<limits.h>`.

All our unsigned-to-unsigned conversions can be performed this way:
test against the upper limit of the result type and abort if the
test fails.

##### Conversion Ranks

There's one question though: why does this test work? In
`val_u16 > UCHAR_MAX`, we compare the value of a 16-bit type and the
value of an 8-bit type. Why does the compiler know how to perform the
comparision?

To perform the *greater-than* operation in the test, the compiler has
to convert both operands to the same type. This controlled by the
types' conversion ranks. The *conversion rank* is a property
of each type in C. It specifies the implicit conversions during
operations, such as the *>* in our test.

Each type has it's own rank, which it only shares with its corresponding
signed or unsigned type. The ranks are ordered in the following way:

 `_Bool` < `unsigned char` < `unsigned short` < `unsigned int` < `unsigned long` < `unsigned long long`.

If an operation receives two operands with different conversion ranks,
the compiler first generates code to that converts the operand with
the lesser rank to the type with the higher rank. The resulting temporary
value is then used for the operation.

This implicit conversion works because of the way the C types are defined.
Larger types always cover the complete range of smaller types:
`unsigned short` covers the complete range of `unsigned char`,
`unsigned int` covers the complete range of `unsigned short`, and so on.
Converting from a lower-rank type to a higher-rank type is therefore safe.

Coming back to our example, the C compiler first converts the value
of `UCHAR_MAX` from `unsigned char` to `unsigned short`, and then
performs the test with the intermediate value.[^1]

#### Case 2: Signed to Signed

Conversions among unsigned types are simple. Luckily, the same rules
apply to *signed* types as well. Conversion ranks among signed types are the
same as for their unsigned counterparts. This means, we can do the same
test for the upper limit of the result type as in the case of *unsigned.*

~~~ c
    short val_s16 = 456;
    signed char val_s8;

    if (val_s16 > SCHAR_MAX) {
        report_error_and_abort(EDOM);
    }
    val_s8 = (signed char)val_s16;
~~~

The constant `SCHAR_MAX` holds the maximum value of `signed char`. For
historical reasons, the type `char` can be signed or unsigned; so it's
best to avoid it here in favor of an explicit `signed char` or
`unsigned char`.

With signed types, we can have negative numbers. The minimum value of
a signed type is stored in a constant named something like `<type>_MIN`.
For `signed char` this is `SCHAR_MIN`. To cover cases where a value is
too low for the result type, we have to extend our test a bit.

~~~ c
    short val_s16 = -456;
    signed char val_s8;

    if (val_s16 < SCHAR_MIN) {
        report_error_and_abort(EDOM);
    } else if (val_s16 > SCHAR_MAX) {
        report_error_and_abort(EDOM);
    }
    val_s8 = (signed char)val_s16;
~~~

We're now also testing against `SCHAR_MIN`, but the overall logic remains
the same.

#### Case 3: Unsigned to Signed

In the case of an converting a value from an unsigned type to a signed
type, we have to make sure that the result is not larger than the maximum
value of the signed type. Our previous code already covers these cases.

~~~ c
    unsigned short val_u16 = 456;
    signed char val_s8;

    if (val_u16 > SCHAR_MAX) {
        report_error_and_abort(EDOM);
    }
    val_s8 = (signed char)val_u16;
~~~

There's really not much difference to the unsigned-to-unsigned case. The
C compiler converts the constant `SCHAR_MAX` to `unsigned short` for
comparing it to `val_u16`. Because *chars* are never larger than *shorts*
the temporary value is always representable by `unsigned short`.

However, if we compile such code, we might get a compiler warning about
mixing signed and unsigned values in the test's condition. This requires
a closer look at how signed and unsigned types are compared.

### Case 4: Signed to Unsigned

Comparing signed and unsigned values is tricky, because unsigned types
cannot represent negative values. Let's take the following example.

~~~ c
    short val_s16 = -1;
    unsigned char val_u8;

    if (val_s16 < 0ull) {
        report_error_and_abort(EDOM);
    } else if (val_s16 > UCHAR_MAX) {
        report_error_and_abort(EDOM);
    }
    val_u8 = (unsigned char)val_s16;
~~~

Here we compare a signed 16-bit value against *0ull* and `UCHAR_MAX`,
which are both unsigned. If we'd compile and run this code, we'd be
surprised by the fact that both tests succeed and that the result
value of `val_u8` is *255.* We would have expected the first test
to fail.

The exact rules for comparing signed and unsigned types in C are
complex, but generally speaking, the compiler picks the operand type
with the highest rank, converts both operands to the unsigned
version of this type, and finally compares these intermediate unsigned
operands.

In the first test, *0ull* is of type `unsigned long long`, which has
a higher rank than `short`. So the compiler converts the value of
`val_s16` to `long long` *and* interprets the resulting bits as
`unsigned long long`. When stored as `long long`, the bit pattern of *-1* is
*11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111,*
assuming 64-bit `long long` values. Interpreting this bit pattern as
unsigned gives a value of *2<sup>64</sup>-1.* And suddenly *-1* is larger
than *0.*

We can fix this problem by comparing against *0ll*, a signed zero,
in the first test.

~~~ c
    short val_s16 = -1;
    unsigned char val_u8;

    if (val_s16 < 0ll) {
        report_error_and_abort(EDOM);
    } else if (val_s16 > UCHAR_MAX) {
        report_error_and_abort(EDOM);
    }
    val_u8 = (unsigned char)val_s16;
~~~

The first test is now a signed-to-signed conversion, which is always
safe to perform. The second test is also safe to perform, as the
range of `short` can represent `UCHAR_MAX`.

#### Putting It All Together

Ideally, we'd have a single function for each conversion, something like
`cast_int_to_ushort()`, `cast_char_to_int()`, `cast_long_to_bool()` and so
on. Even better, if we could have a generator macro that creates these
functions for us.

The blog post is getting rather long already, so please forgive that I'm
not going to explain all the details of how to do this. Rather, I'd like
to point you to the [picotm source code][picotm:picotm_cast_tx], which
contains exactly this.

The C pre-processor macro `PICOTM_CAST_TX()` generates conversion functions
from one C type to another C type. For example, running

~~~ c
PICOTM_CAST_TX(uint, unsigned int, short, short, SHRT_MAX, SHORT_MIN)
~~~

produces

~~~ c
static inline short
cast_uint_to_short_tx(unsigned int value);
~~~

which safely casts any value of type `unsigned int` to type `short`. The
function detects the signess of the input and result types, makes sure that
it's not comparing values of signed and unsigned types in the wrong places,
and reports errors.

A lot of the analysis depends on the involved types and can be done by
the C compiler while building the program. An optimizing compiler can easily
eliminate most of the cases. For example, if both types have the same
signess, all the cases for handling differences in signess are removed.

As picotm provides system-level transactions for safe and reliable
programming, errors are reported to the picotm transaction manager, which
will roll-back the transaction and run the error-recovery code. But this is
entirely separated and unrelated to the type conversion.

#### Summary

In this blog post, we've looked at the different cases of converting
integer types among each other. This includes

 - unsigned to unsigned,
 - signed to signed,
 - unsigned to signed, and
 - signed to unsigned.

There are a lot of corner cases and the blog post covers all of the tricky
parts. Finally, we've seen a generator macro that creates conversion
functions during the C compiler's pre-processing stage. Using such a macro,
a large number of safe type-casting functions can be created easily.

If you like this blog post, please subscribe to the RSS feed, follow on
Twitter or share on social networks.

#### Footnotes

[^1]:   C compilers don't have to follow this procedure step-by-step. A
        compiler is allowed to optimize the code as long as the result is
        the same as in the non-optimized variant.

[picotm]:                   http://picotm.org/
[picotm:picotm_cast_tx]:    http://github.com/picotm/picotm/blob/master/modules/cast/include/picotm/picotm-cast.h#L66
