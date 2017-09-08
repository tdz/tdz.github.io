---
layout:     post
title:      malloc()'s Tricky Error Reporting
date:       2017-09-08 12:30:00 +0200
tags:       [c, errno, free, howto, linux, malloc, posix, tutorial, unix]
og-image:   tux.png
---

Most error checks for `malloc()` are incorrect. In this blog post, we're
going to look at the details of `malloc()`'s error reporting semantics and
how to test if a call to `malloc()` succeeded.

<!-- excerpt -->

#### The Semantics of `malloc()`

If you have seen at least a few lines of C code, you've probably come
across [`malloc()`][posix:malloc], the interface for allocating memory
from the operating system at runtime.

``` c
size_t size = calc_amount_of_bytes();

void* ptr = malloc(size);
```

In this example, the function `calc_amount_of_bytes()` is a placeholder
that returns the number of bytes to be allocated. This memory size is given
to `malloc()`, which then acquires a large-enough memory buffer from the
operating system. Finally, `malloc()` returns a pointer to the beginning
of the buffer.

#### First Try: Testing the Returned Value

In the case that `malloc()` failed to allocate the requested amount
of memory, it returns a `NULL` pointer. Quoting
[`malloc()`'s man page][man7:malloc]:

> The  malloc()  and  calloc()  functions return a pointer to the
> allocated memory, which is suitably aligned for any built-in type.
> On error, these functions return NULL.

Similar descriptions can be found in the ISO C Standard, the POSIX
Standard and the `malloc()` page on MSDN.

The usual, almost ideomatic way of testing for allocation errors therefore
looks like this.

``` c
size_t size = calc_amount_of_bytes();

void* ptr = malloc(size);
if (!ptr) {
    // allocation error detected!
}
```

And this code is... **wrong.**

Again, here's what `malloc()`'s man page says. Note the additional sentence.

> The  malloc()  and  calloc()  functions return a pointer to the allocated
> memory, which is suitably aligned for any built-in type.  On error, these
> functions return NULL.  NULL may also be returned by a successful call to
> malloc() with a size of zero [...].

So if the allocation size is *0,* `malloc()` is allowed to return `NULL` even
though it actually succeeded. For some reason, everyone seems to ignore this
additional sentence. I admit that I'm guilty of this as well.

#### Second Try: Testing `errno`

Just testing the returned pointer for `NULL` doesn't work. My solution to
this problem was to test the value of [`errno`][posix:errno] instead. If
the allocation fails, `malloc()` sets the value of `errno` to `ENOMEM`
before it returns `NULL`.

The pattern for testing `malloc()` errors via `errno` looks like this.

``` c
size_t size = calc_amount_of_bytes();

errno = 0;
void* ptr = malloc(size);
if (errno) {
    // allocation error detected!
}
```

Here, we clear `errno` to a neutral value before calling `malloc()`. If
`errno` is set afterwards, an allocation error has accured.

Unfortunately, it's not a correct solution either. The pattern worked
very well on all kinds of Linux systems until I came across a `malloc()`
implementation on one of Sony's Android phones.

On Firefox OS, we used to have a number of system daemons for running the
Android drivers separately from the main system. When reviewing patches to
these daemons, I asked patch authors to use the `errno` pattern and avoid
the test for the `NULL` pointer.

On Sony's Android phone, this didn't work. We had this pattern in our
Bluetooth daemon, but Sony's implementation of `malloc()` sets `errno` even
if the allocation succeeds. The Bluetooth daemon incorrectly interpreted
this as an error.

I don't blame Sony for this problem. It was a mistake on my side, as the
POSIX standard says the following about `errno`:

> No function in this volume of POSIX.1-2008 shall set errno to 0. The
> setting of errno after a successful call to a function is unspecified
> unless the description of that function specifies that errno shall not
> be modified.

In other words, even in the case of success, `malloc()` is allowed to modify
the value of `errno`. It's just not allowed to set it to *0.*

#### Third Try: Testing the Allocation Size and the Returned Pointer

This brings us back to testing the returned pointer. Since it's only
allowed to be legally `NULL` if the requested size is zero, we have to
test specifically for this combination. This pattern looks as shown below.

``` c
size_t size = calc_amount_of_bytes();

void* ptr = malloc(size);
if (size && !ptr) {
    // allocation error detected!
}
```

And this is the solution to the problem. The pointer test is only allowed
if we requested a memory buffer with a size greater than zero. And only then
it's an error. Allocations of zero-size buffers always succeed.

It's important to note that the behavior for zero-size allocations is
completely up to the implementation. It's also legal that `malloc()` returns
a pointer different from `NULL`. It could even set `errno` in this case. Our
current pattern handles these implementations as well.

A nice detail about releasing the allocated memory with
[`free()`][posix:free] is, that `free()` accepts `NULL` as a valid argument.
So it's always possible to use a code pattern like the one shown below.

``` c
size_t size = calc_amount_of_bytes();

void* ptr = malloc(size);
if (size && !ptr) {
    // allocation error detected!
}

free(ptr);
```

If you know a recent system where this code fails for allocation of zero
size, I'd like to [hear about][mail:tdz] it.

#### Summary

In this blog post, we investigated different patterns for testing
for failed allocations.

 - `malloc()` returns `NULL` on errors.
 - `malloc()` may return `NULL` for zero-size allocations.
 - Testing the returned value does not work reliably with zero-size
   allocations.
 - `malloc()` is allowed to modify `errno` during successful allocations.
 - Testing the returned error code does not work reliably.
 - Testing the returned value iff the allocation size is larger than zero
   works reliably on modern systems.
 - A `NULL` pointer returned by a zero-size allocation is accepted by
   `free()`.

If you like this blog post about memory allocation, please subscribe to
the RSS feed, follow on Twitter or share on social networks.

[mail:tdz]:                 mailto:{{ site.author.email }}
[man7:malloc]:              http://man7.org/linux/man-pages/man3/malloc.3.html
[posix:errno]:              http://pubs.opengroup.org/onlinepubs/9699919799/functions/errno.html
[posix:free]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/free.html
[posix:malloc]:             http://pubs.opengroup.org/onlinepubs/9699919799/functions/malloc.html
