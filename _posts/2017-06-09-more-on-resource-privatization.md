---
layout:     post
title:      More on Resource Privatization
date:       2017-06-09 11:00:00 +0200
tags:       [c, howto, memcpy, nosql, posix, privatization, stm, transaction, transactional memory, tutorial]
og-image:   share2.png
---

This is a follow-up post to [last week's piece][post20170601] on resource
privatization for transactional memory. We're going to support C strings and
look at combining load and store operations with privatization.

<!-- excerpt -->

#### Privatizing C Strings

Privatazion provides a way of directly accessing a resource from within
a transaction. For memory this means direct access to the memory region,
instead of working on a transaction-local copy.

The interface for privatizing regions of memory looks like this.

~~~ c
void
privatize(uintptr_t addr, size_t siz, bool load, bool store);
~~~

This function takes the address of the first byte of the memory region to
be privatized, the number bytes in the region, and two booleans enabling
load and store access. If it succeeds, the memory region is owned by the
transaction and can be accessed directly.

The interface is fine for memory regions of known size. In last week's
example code we implemented a transactional [`memcpy()`][posix:memcpy]; and
anything that is typically accessed with `memcpy()` by be privatized this
way.

There is one noteable exception: C strings. Strings in C have their length
implicitly signalled with a trailing *\0* character. The get a string's
length a call to [`strlen()`][posix:strlen] is required.

A first attempt to privatize a C string will fail quickly.

~~~ c
char* str = "Hello World!";

tm_begin

    privatize(str, ???, true, false);

tm_commit
~~~

When using the current interface, we don't know of which length the string
is. Maybe using `strlen()` could help.

~~~ c
char* str = "Hello World!";

tm_begin

    size_t siz = strlen(str);
    privatize(str, siz, true, false);

tm_commit
~~~

We now have all parameters required by `privatize()` but the call to
`strlen()` does not protect against concurrent access to `str`. Another
transaction can modify the content of `str` concurrently, which would
make the returned length incorrect. Maybe we can solve this problem by
providing a transactional implementation of `strlen()`.

~~~ c
size_t
strlen_tx(const char* str)
{
    privatize(str, ???, true, false);

    return strlen(str);
}

char* str = "Hello World!";

tm_begin

    size_t siz = strlen_tx(str);
    privatize(str, siz, true, false);

tm_commit
~~~

This approach creates a chicken-and-egg problem. Our transactional
implementation of `strlen()` requires `privatize()` to protect the
string against concurrent modification. But `privatize()` is what we're
trying to support in the first place.

Clearly a different approach is required.

The solution implemented by [picotm][picotm] is to have a privatize function
that stops at a caller-specified terminating character. For a C string the
terminating character would be *\0.*

Our function is called `privatize_c()` with `c` for character. The interface
is similar to the regular `privatize()`, but it receives a character instead
of the length.

~~~ c
void
privatize_c(uintptr_t addr, int chr, bool load, bool store);
{
    bool found_chr = false

    while (!found_chr) {

        struct resource* res = acquire_resource(addr & BASE_BITMASK);
        if (!res) {
            tm_restart();
        }

        unsigned long index = addr & RESOURCE_BITMASK;
        unsigned long bits = 1ul << index;

        uint8_t* beg = arraybeg(res->local_value) + index;
        uint8_t* end = arrayend(res->local_value);

        while (!found_chr && (beg < end)) {
            /* If we're about to store, we first have to
             * save the old value for possible rollbacks. */
            if (store && !(res->local_bits & bits) ) {
                *beg = *((uint8_t*)addr);
            }

            bits <<= 1;
            --siz;
            ++addr;
            ++beg;

            found_chr = (*beg == chr);
        }

        if (store) {
            res->flags |= RESOURCE_FLAG_WRITE_THROUGH;
        }
    }
}
~~~

The code is again very similar to the implementation of the original
`privatize()`, but instead of testing for the size of the supplied memory
region, it tests for the existence of the terminating character. Once the
memory location containing the character has been privatized, `privatize_c()`
returns.

Our original example now becomes trivial to implement.

~~~ c
char* str = "Hello World!";

tm_begin

    privatize_c(str, '\0', true, false);

tm_commit
~~~

#### Loading and Storing Privatized Memory

Remember that load and store operations used a transaction-local buffer
to work on. We called this *write-back,* as the buffer is only written back
to the shared memory resource during a commit. Privatization instead required
*write-through* semantics, where transactions operate directly on the shared
resource.

Because of this difference, our implementation of `privatize()` can not be
combined with `load()` and `store()` calls on the same memory regions. Here's
an example transaction.[^1]

~~~ c
int value = 0;

tm_begin

    privatize(&value, sizeof(value), true, true);

    value = 1;

    load_int(&value); /* returns 0 */

tm_commit
~~~

This transaction privatizes an integer variable and stores *1.* It then
transactionally loads the variable, but this gives an incorrect result of
*0.*

Loads and stores currently use a write-back scheme. When acquiring a
resource, they operate on a transaction-local buffer. Only at commit
time this buffer is written back to the shared value. The call to
`privatize()` initializes the transaction-local buffer with the original
value of *0,* so this is what `load_int()` returns.

The solution to this problem is to support write-through mode for load
and store operation. Without going into details, an implementation would have
to provide the following semantics.

 * For initial loads and stores, acquire the resource and use write-back.
 * For initial privatize, acquire the resource and use write-through.
 * For load and stores on an already-privatized resource, operate on the
   shared buffer, instead of the transaction-local buffer.
 * For privatizing an already-loaded or already-stored resource, swap the
   content of transaction-local and shared buffer, and set write-through
   mode for the resource.

This behavior is implemented in [picotm][picotm], but probably not worth
the effort for our *simpletm* transaction manager.

A further optimization would be the use of write-through mode for all load
operations until the transaction executed a store operation on the resource.
This would spare reader transactions from the overhead of filling
transaction-local buffers that they are never going to modify.

#### Summary

In this block post, we looked at two corner cases of memory privatization.

 - Memory regions with terminating character, such as C strings, can be
   privatized by stopping privatization at the specific character.
 - Load and store operations can be combined with privatization by supporting
   write-through semantics when loading and storing shared resources.

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

#### Footnotes

[^1]:   There might be other limitations in our current implementation
        that prevent this example from working. One is that acquiring the
        same resource multiple times from within the same transaction is
        currently not supported.


[picotm]:       http://picotm.org/
[posix:memcpy]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/memcpy.html
[posix:strlen]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/strlen.html
[post20170601]: {% post_url 2017-06-01-transactional-memcpy-and-resource-privatization %}
