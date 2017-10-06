---
layout:     post
title:      When realloc() Doesn't Allocate
date:       2017-10-06 08:00:00 +0200
tags:       [c, errno, howto, linux, malloc, posix, realloc, tutorial, unix]
og-image:   tux.png
---

I recently wrote about [correct error handling for `malloc()`][post:20170908].
Coincidently I came across an older [defect report][openstd:dr400] on the
behavior of C's [`realloc()`][man7:realloc] function just a few days ago. In
this blog post, we're going to look at `realloc()`'s behavior if it's out
of memory or if the requested size is zero.

<!-- excerpt -->

#### A Simple `realloc()`

In C11, new memory blocks are dynamically allocated using one of
[`malloc()`][man7:malloc], [`calloc()`][man7:calloc], or
[`aligned_alloc()`][man7:aligned_alloc].

If you read this blog post, you've probably seen calls to these functions
frequently, including calls to [`realloc()`][man7:realloc]. A call to
`realloc()` changes the size of a memory block while preserving the block's
existing content. Here's a naive implementation.

``` c
void*
realloc(void* old_mem, size_t new_size)
{
    if (!old_mem) {
        return malloc(new_size);
    } else if (!new_size) {
        return old_mem;
    }

    void* new_mem = malloc(new_size);
    if (!new_mem) {
        return NULL;
    }

    size_t old_size = ; // non-portable: get size of 'old_mem' from allocator

    memcpy(new_mem, old_mem, old_size < new_size ? old_size : new_size);
    free(old_mem);

    return new_mem;
}
```

What does it do? First of all, we get two corner cases out of the way. If
no old memory buffer is specified, `realloc()` shall behave like `malloc()`.
Our first condition handles this. Then we test if the new size is not zero. If
it is zero, we return the old memory. [C11 with DR 400][openstd:dr400] say that
successful zero-size allocations shall behave *as if the size were some nonzero
value [...].* We already know that `old_mem` contains a non-`NULL` value, so we
return this. It's not supposed to be dereferenced after this point, though.

With the corner cases handled, we can perform the *real* `realloc()`, which is
`malloc()`-`memcpy()`-`free()` semantically. And finally we return the new
buffer, which contains the content of the old buffer.

For the `memcpy()` operation, we need the size of the old buffer. There's no
such functionality standardized by ISO C or POSIX, but most allocators provide
an interface for this: [`malloc_usable_size()`][man7:malloc_usable_size] in the
GNU libc, BSD and Bionic, [`_msize()`][msdn:_msize] on Windows, or
[`malloc_size()`][unix:malloc_size] on MacOS.

#### Handling the Returned Value

Let's now call `realloc()` and see what it does. One common programming
mistake is illustrated below.

``` c
void* mem = malloc(10); /* non-zero allocation */
...

mem = realloc(mem, 20);
if (!mem) {
    /* handle allocation failure */
}
```

This pattern can often be found, although it is incorrect. What happens if
the re-allocation fails? Of course, `NULL` is returned. If we immediately
store `realloc()`'s returned value in `mem`, the old `mem` buffer leaks if
`realloc()` fails. The correct code looks like this.

``` c
void* mem = malloc(10); // non-zero allocation
...

void* tmp = realloc(mem, 20);
if (!tmp) {
    /* handle allocation failure */
}
mem = tmp; /* Only update 'mem' is it's safe to do so */
```

Even if `realloc()` failed to allocate a new buffer, we still have our
pointer to the old buffer and can free the old buffer later on.

Another, although less common, mistake is to access memory out-of-bounds
after shrinking a buffer. Here's an example.

``` c
char* mem = malloc(10); /* non-zero allocation */
...

void* tmp = realloc(mem, 5); /* shrink buffer */
if (!tmp) {
    /* handle allocation failure */
}
mem = tmp;

mem[7] = 0; /* This memory location is now outside the allocated buffer! */
```

Remember that semantically `realloc()` allocates a new buffer, copies the
old buffer into the new buffer, and then frees the old buffer. Depending on
the requested buffer size, the actual buffer size can also shrink. Accessing
memory that was allocated with the old buffer, but not with the new buffer is
therefore undefined behavior.

Finally, let's see what happens if the requested new size is zero. Quoting
[DR 400][openstd:dr400] again:

|       | `realloc(NULL, 0)`                         | `realloc(non-NULL ptr, 0)`                                |
| ----- | ------------------------------------------ | --------------------------------------------------------- |
| AIX   | always returns `NULL`, errno is `EINVAL`   | always returns `NULL`, frees `ptr`, errno is `EINVAL`     |
| BSD   | returns `NULL` on error, errno is `ENOMEM` | returns NULL on error, `ptr` unchanged, errno is `ENOMEM` |
| glibc | returns `NULL` on error, errno is `ENOMEM` | always returns `NULL`, `ptr` freed, errno unchanged       |

This is very inconsistent among older C implementations before C11.
Every implementation behaves in a different way. I can't bring myself
to even try to write error-checking code that works correctly with all
these variants. If you do, please [send me some example code][mail:tdz]
and I'll post it here.

If we're on C11, we're lucky. For zero-size reallocations, DR 400
specifies that `realloc()` shall either return a `NULL` pointer to indicate
an error, or that the old buffer remains in place. For all pre-C11 code that
requires portability between different implementations, it's best to
write a wrapper around `realloc()` that handles the corner cases specifically.

``` c
void*
portable_realloc(void* mem, size_t siz)
{
    if (!mem) {
        return malloc(siz);
    } else if (!siz) {
        return mem;
    }
    return realloc(mem, siz);
}
```

#### Summary

In this blog post we looked at the corner cases of `realloc()`.

 - If the old memory buffer is `NULL`, `realloc()` behaves like `malloc()`.
 - If the requested size is zero, behavior is implementation defined.
 - On allocation errors, pre-C11 implementations of `realloc()` handle
   `errno`, the old memory buffer and their return value in different ways.
 - Portable code should either switch to C11, or wrap `realloc()` in a
   function that handles the corner cases in a portable way.
 - Callers must hold a pointer to the old buffer until they tested for
   the success of `realloc()`. Otherwise the application leaks the old
   memory buffer on allocation failures.
 - Calls to `realloc` may shrink memory buffers. Memory that was present
   in the old buffer, may not be available in the new shrinked memory buffer.

If you like this blog post about memory allocation, please subscribe to
the RSS feed, follow on Twitter or share on social networks.

[mail:tdz]:                 mailto:{{ site.author.email }}
[man7:aligned_alloc]:       http://man7.org/linux/man-pages/man3/aligned_alloc.3.html
[man7:free]:                http://man7.org/linux/man-pages/man3/free.3.html
[man7:calloc]:              http://man7.org/linux/man-pages/man3/calloc.3.html
[man7:malloc]:              http://man7.org/linux/man-pages/man3/malloc.3.html
[man7:malloc_usable_size]:  http://man7.org/linux/man-pages/man3/malloc_usable_size.3.html
[man7:realloc]:             http://man7.org/linux/man-pages/man3/realloc.3.html
[msdn:_msize]:              http://msdn.microsoft.com/en-us/library/z2s077bc(v=vs.140).aspx
[openstd:dr400]:            http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2109.htm#dr_400
[post:20170908]:            {% post_url 2017-09-08-mallocs-tricky-error-reporting %}
[unix:malloc_size]:         http://www.unix.com/man-page/osx/3/malloc_size/
