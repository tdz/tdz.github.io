---
layout:     post
title:      The Strange strerror_r() of Dr POSIX and Mr GNU
date:       2017-08-25 10:00:00 +0200
tags:       [c, error handling, glibc, gnu, linux, strerror_r, transactions, unix]
og-image:   gnu-head.png
---

There are several variants of the C function
[`strerror_r()`][posix:strerror_r] that differ in their return value and
error handling. This blog post describes how to support all of them in a
transactional interface, while still being compatible with either internal
implementation. As such, `strerror_r()` serves as a case study for
transactional interfaces with multiple or variable semantics.

<!-- excerpt -->

This blog post builds on previous entries about transactional C code and
system-level transactions. If you missed earlier entries in the
[series][post:20170505],
[you][post:20170509]
[might][post:20170512]
[want][post:20170519]
[to][post:20170526]
[go][post:20170601]
[back][post:20170609]
[and][post:20170616]
[read][post:20170623]
[the][post:20170630]
[installments][post:20170707]
[so][post:20170714]
[far][post:20170721].

#### The Old, Single-threaded Times

The original C `strerror()` function returns a descriptive string for
each [`errno`][posix:errno] constant, as shown below.

``` c
char* str = strerror(ENOMEM);
printf("%s\n", str); // prints "Out of memory"
```

The example retrieves and prints a description of the errno code `ENOMEM`,
which results in an output like `Out of memory` or a similar text.

This works well in a single-threaded program, but fails in multi-threaded
software. `strerror()` is allowed to use an internal buffer for constructing
and storing the returned string. The buffer is shared among all threads, so
if multiple threads execute the function at the same time, they overwrite
each other's error strings.

#### The Multi-threaded Interface

The function `strerror_r()` is a variant of `strerror` that is suitable for
multi-threaded environments. As typical for thread-safe interfaces of the C
standard library, `strerror_r()` takes an additional argument that provides
thread-local memory for its internal operations. In this case it's a string
buffer and the buffer length.

Rewriting our previous example with `strerror_r()` we get the following
code.

``` c
char buf[256]; // large-enough string buffer
char* str = strerror_r(ENOMEM, buf, sizeof(buf));
printf("%s\n", str); // prints "Out of memory"
```

Any temporary error string is stored in the supplied memory of `buf`, which
is owned by the current thread.

#### The GNU Interface

The function `strerror_r()` in the example returns a pointer to the error
string, which is possibly stored in the memory of `buf`. This is the [GNU
definition of `strerror_r()`][gnu:strerror_r].

The GNU variant returns an internal string for known errors, or copies
something like `Unknown error code` into the provided buffer and returns
the buffer's address. In any case no error is returned.

#### The POSIX Interface

The GNU interface probably existed before `strerror_r()` was standardized
by POSIX in 2001. Unfortunately the POSIX interface works differently. Here
again our example.

``` c
char buf[256]; // large-enough string buffer
int err = strerror_r(ENOMEM, buf, sizeof(buf));
if (err) {
    // handle error stored in err
}
printf("%s\n", buf); // prints "Out of memory"
```

The variant of `strerror_r()` as standardized by POSIX returns an integer
value that signals the success or failure of the operation. If `strerror_r()`
succeeds, it returns a value of *0.* The provided buffer now contains the
error string. If it returns the constant `EINVAL`, the provided `errno`
constant is unknown and the buffer is still empty. If `strerror_r()` returns
the constant `ERANGE`, the provided buffer space is too small to store the
error string and the caller is supposed to provide a larger buffer.

#### The Semi-POSIX Interface in Older glibc

In the GNU C library before release 2.13, there exists yet another variant
of `strerror_r()`. It's similar to the POSIX variant, but returns `-1` on
errors and stores the error code in `errno`. Using that variant, our example
looks like this.

``` c
char buf[256]; // large-enough string buffer
int res = strerror_r(ENOMEM, buf, sizeof(buf));
if (res < 0) {
    // handle error stored in errno
}
printf("%s\n", buf); // prints "Out of memory"
```

I tried to find out more about the origins of this variant, but was not
successful so far. I guess that the use of different semantics were either
an oversite, or it was part of some other standard or system that pre-dates
POSIX 2001.

#### The Transactional Interface

For using `strerror_r()` from within transactions, we have to provide a
transaction-safe wrapper function. This function has to protect all
input parameters from concurrent access and handle returned errors by
starting the transaction's recovery code.

Let's start with providing a transactional interface that users can call. In
the GNU C library, which we'll use for the rest of this discussion, all
variants of `strerror_r()` exist in one way or the other. To make them work,
we create two internal interfaces; one for GNU and one for POSIX.

``` c
char*
__strerror_r_gnu_tx(int errnum, char* buf, size_t buflen);

int
__strerror_r_posix_tx(int errnum, char* buf, size_t buflen);
```

Which one is used by the caller depends on the compile-time flags of the
caller's program. The POSIX declararion is provided by defining
`_POSIX_C_SOURCE >= 200112L)` and undefining `_GNU_SOURCE` in the
C preprocessor. This means that if we follow POSIX *and* disallow GNU
extenions, we get the POSIX variant, otherwise we get the GNU variant.

Our transactional `strerror_r()` is called `strerror_r_tx()`. We provide it
as a C pre-processor macro that switches between the variants.

``` c
#if (_POSIX_C_SOURCE >= 200112L) && !_GNU_SOURCE
#define strerror_r_tx   __strerror_r_posix_tx
#else
#define strerror_r_tx   __strerror_r_gnu_tx
#endif
```

This was easy. If we follow POSIX strictly, a call to `strerror_r_tx()`
will result in invoking `__strerror_r_posix_tx()`, otherwise
`__strerror_r_gnu_tx()`.

#### The Transactional GNU Implementation

We implement the transactional GNU and POSIX interfaces with either
`__strerror_r_posix_tx()` or `__strerror_r_gnu_tx()`. These functions are
part of the transaction system and we always have to provide both because
we don't know if the caller's program is build against one or the other.
Depending on the transaction system's compile-time flags, either function's
implementation could in turn use the GNU or the POSIX implementation
internally.

For simplicity, let's strip down the example code to the minimum. Here's the
transactional GNU implementation.

``` c
char*
__strerror_r_gnu_tx(int errnum, char* buf, size_t buflen);
{
    // protect buf from concurrent access here

    // save errno here

    char* str;

#if (_POSIX_C_SOURCE >= 200112L) && !_GNU_SOURCE

    // POSIX

    char tmpbuf[256]; // large-enough buffer for all error strings

    int res = strerror_r(errnum, tmpbuf, sizeof(tmpbuf));

    // TODO: error handling

    if (buflen) {
        size_t tmplen = strlen(tmpbuf);
        str = memcpy(buf, tmpbuf, MIN(tmplen + 1, buflen));
        str[MIN(tmplen, buflen - 1)] = '\0';
    }

#else

    // GNU

    str = strerror_r(errnum, buf, buflen);

#endif

    return str;
}
```

In the example, we first protect the input buffer against concurrent access
and save the state of `errno`. Other modules of the transactional system do
this for us, so it's left out here. If you're curious, previous entries in
this blog explain how it works.

If we implement `__strerror_r_gnu_tx` with the GNU variant, there's nothing
we have to do. Just wrap the function.

If we implement `__strerror_r_gnu_tx` with the POSIX variant, we execute it
with the temporary buffer `tmpbuf` and copy the result into the
caller-provided buffer. The point of using a temporary buffer is to avoid
`strerror_r()` return the error code `ERANGE`. For this reason the memory
in `tmpbuf` must be large enough to hold *any* string returned by POSIX'
`strerror_r()`. When we later copy the temporary buffer to the output buffer,
we can cut-off any trailing bytes that do not fit into the output buffer.

There's one TODO comment left in the example. It's the error handling for
`strerror_r()`. Let's add this.

First of all, there are three cases to handle here. A return value of *0*
always signals success. A negative return value signals an error with glibc
before 2.13 and the error is stored in `errno`. A positive return value
signals an error with glibc 2.13 and later, and the error is the returned
value itself.

Second, we can handle the error code of `EINVAL`, which signals an unknown
error code, by manually creating an appropriate string. The error code
`ERANGE`, which signals that the provided buffer is too small, is avoided
by choosing the size of `tmpbuf` accordingly. If we see any other error,
we have to run the transaction's error recovery, as required by transactional
semantics.

Let's fill in the error handling.

``` c
static void
handle_errnum(int errnum, char* buf)
{
    if (errnum == EINVAL) {
        memcpy(buf, "Unknown error code", sizeof("Unknown error code"));
    } else {
        // invoke transactional error recovery here
    }
}

char*
__strerror_r_gnu_tx(int errnum, char* buf, size_t buflen)
{
    // protect buf from concurrent access here

    // save errno here

    char* str;

#if (_POSIX_C_SOURCE >= 200112L) && !_GNU_SOURCE

    // POSIX

    char tmpbuf[256]; // large-enough buffer for all error strings

    int res = strerror_r(errnum, tmpbuf, sizeof(tmpbuf));

    if (res < 0) { // glibc < 2.13
        handle_errnum(errno, tmpbuf);
    } else if (res) { // glibc >= 2.13
        handle_errnum(res, tmpbuf);
    }

    if (buflen) {
        size_t tmplen = strlen(tmpbuf);
        str = memcpy(buf, tmpbuf, MIN(tmplen + 1, buflen));
        str[MIN(tmplen, buflen - 1)] = '\0';
    } else {
        str = buf;
    }

#else

    // GNU

    str = strerror_r(errnum, buf, buflen);

#endif

    return str;
}
```

#### The Transactional POSIX Implementation

After building the transactional GNU implementation, we need a similar
implementation for the POSIX variant `__strerror_r_posix_tx`. This time we
implement the POSIX declaration with either the GNU implementation or the
POSIX implementation itself. Therefore we have to return an integer value.

There's one interesting corner case here. While `ERANGE`, the buffer-too-small
signal, is technically an error, it's not actually used as such. A transaction
might want to call the function with an initial buffer, check for `ERANGE`, and
grow the buffer until the error string fits in. So `ERANGE` is not an error,
but a status code. That's a valid pattern and our transactional implementation
should be able to support it.

``` c
int
__strerror_r_posix_tx(int errnum, char* buf, size_t buflen)
{
    // protect buf from concurrent access here

    // save errno here

#if (_POSIX_C_SOURCE >= 200112L) && !_GNU_SOURCE

    // POSIX

    int res = strerror_r(errnum, buf, buflen);

    if (res < 0) { // glibc < 2.13
        if (errno == ERANGE) {
            return res;
        } else {
            // invoke transactional error recovery here
        }
    } else if (res) { // glibc >= 2.13
        if (res == ERANGE) {
            return res;
        } else {
            // invoke transactional error recovery here
        }
    }

#else

    // GNU

    char tmpbuf[256];

    char* str = strerror_r(errnum, tmpbuf, tmplen);
    if (str == tmpbuf) {
        // invoke transactional error recovery for EINVAL
    }

    if (strlen(str) + 1 > buflen) {
        if (glibc version >= 2.13) {
            return ERANGE;
        } else {
            errno = ERANGE;
            return -1;
        }
    }

    mempcy(buf, str, strlen(str) + 1);

#endif

    return 0;
}
```

As before, we first protect the input input buffer against concurrent access
and save the value of errno.

If the first case, where we use the POSIX implementation of `strerror_r()`,
we check the return value for possible errors. As before, we have to handle
different versions of the glibc differently. If we see an error of type
`ERANGE`, we return it to the calling transaction. That's the case were the
error is actually a status code. The other possible error, `EINVAL`, is
actually an error and triggers transactional error recovery.

In the second case, we use the GNU implementation of `strerror_r()`. As
mentioned before, the GNU code does not return errors at all. To detect
`ERANGE` and `EINVAL`, we need other means.

The GNU implementation has an interesting property that we can use to our
advantage: it only uses the provided buffer for unknown error codes. For every
known error code it returns a pointer to an internal static string. In our
example we call the GNU implementation with a temporary buffer. If the
returned string pointer has the same address as the temporary buffer, GNU's
`strerror_r()` generated an `Unknown error code` message. In this case we
invoke the transaction's recovery code as if we had seen `EINVAL`. If the
addresses differ, we got an internal static string, so the error code is a
known one.

The other error, `ERANGE`, is detected by comparing the size of the provided
buffer with the length of the error string. If the error string does not fit
into the buffer, we return `ERANGE` to the calling transaction. The
transaction can now resize the buffer and try again.

#### Summary

In blog post, we examined the details of implementing a transactional variant
of `strerror_r()`, a C function with varying interfaces and semantics.[^1] It
serves as a case study for code with similar properties.

 - `strerror_r()` generates an error string for a given `errno` node. It
   possibly stores the string in a caller-provided output buffer.
 - The GNU implementation returns a pointer to the error string.
 - The POSIX implementation return an integer signalling errors and stores
   the string in the provided buffer.
 - The semi-POSIX impementation returns an integer signalling success or
   failure and stores the error code in `errno`.
 - Depending on their compile-time flags, users may call the GNU or POSIX
   variant. Therefore we have to provide a transactional implementation for
   each.
 - The transactional implementations have to ensure compatiblity with the
   semantics of the non-transactional implementations.
 - Some error codes, such as ERANGE, might not be errors, but allow for
   specific programming patterns. The transactional implementations have
   to support these corner cases.

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

#### Footnotes

[^1]:   To add a personal opinion here, the only useful variant of these
        interfaces is the one specified by GNU. When we call `strerror_r()`
        we're probably already in the middle of handling an error. The last
        thing we want is to deal with additional errors coming from the
        error-reporting code. To me, returning a static error string or
        whatever fits into the provided output buffer is acceptable and
        sane semantics for `strerror_r()`.

[gnu:strerror_r]:           https://www.gnu.org/software/libc/manual/html_node/Error-Messages.html#index-strerror_005fr
[posix:errno]:              http://pubs.opengroup.org/onlinepubs/9699919799/functions/errno.html
[posix:strerror]:           http://pubs.opengroup.org/onlinepubs/9699919799/functions/strerror.html
[posix:strerror_r]:         http://pubs.opengroup.org/onlinepubs/9699919799/functions/strerror_r.html
[post:20170505]:            {% post_url 2017-05-05-implementing-a-simple-transaction-manager-in-c %}
[post:20170509]:            {% post_url 2017-05-09-transactional-semantics-in-theory-and-practice %}
[post:20170512]:            {% post_url 2017-05-12-transaction-roll-back-and-serializability %}
[post:20170519]:            {% post_url 2017-05-19-writing-the-beginning-and-end-of-a-transaction %}
[post:20170526]:            {% post_url 2017-05-26-transactional-access-to-arbitrary-memory-locations %}
[post:20170601]:            {% post_url 2017-06-01-transactional-memcpy-and-resource-privatization %}
[post:20170609]:            {% post_url 2017-06-09-more-on-resource-privatization %}
[post:20170616]:            {% post_url 2017-06-16-transaction-logs %}
[post:20170623]:            {% post_url 2017-06-23-transactional-malloc-and-free %}
[post:20170630]:            {% post_url 2017-06-30-everything-about-errno-plus-transactions %}
[post:20170707]:            {% post_url 2017-07-07-types-of-transactional-operations %}
[post:20170714]:            {% post_url 2017-07-14-building-fault-tolerant-software-with-transactions %}
[post:20170721]:            {% post_url 2017-07-21-implementing-fault-tolerant-software-with-transactions %}
