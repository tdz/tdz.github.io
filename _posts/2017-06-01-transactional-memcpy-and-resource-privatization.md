---
layout:     post
title:      Transactional memcpy() and Resource Privatization
date:       2017-06-01 11:00:00 +0200
tags:       [c, howto, memcpy, nosql, posix, privatization, stm, transaction, transactional memory, tutorial]
og-image:   share2.png
---

So far we have [implemented a simple transaction manager][post20170519]
that [loads and stores values in shared memory locations][post20170526].
These load and store operations copy values in and out of the transactional
context. In this blog post we're going to implement direct access to the
shared memory. We also lay the foundation for operations besides memory
access, such as function calls.

<!-- excerpt -->

For our transaction manager, we created an implementation that loads and
stores values by copying them in and out of the transaction context. The
interface is shown below.

~~~ c
void
load(uintptr_t addr, void* buf, size_t siz);

void
store(uintptr_t addr, const void* buf, size_t siz);
~~~

The function `load()` takes an address in main memory and copies `siz`
bytes into a transaction-local buffer. The function `store()` does a
similar thing by copying `siz` bytes from the transactional buffer to
an arbitrary address in memory.

This interface is good for most use cases, but sometimes we need access
to the *actual* shared resource. When we acquire a resource for direct
access, it becomes part of the transaction's private context. We call this
*privatization.*[^1]

#### Why Privatize Resources?

Imaging we have a non-transactional function that takes a memory location
for it's arguments, let's say [`memcpy()`][posix:memcpy]. If we what to call
`memcpy()` from within a transaction, we require an implementation that can
handle transactional memory.

Here's a very simple implementation based on the load and store primitives
we've developed so far.

~~~ c
void*
memcpy_tx(void* dst, const void* src, size_t siz)
{
    uint8_t* dst8 = dst;
    const uint8_t* src8 = src;

    while (siz) {

        uint8_t value;

        load(src8, &value, 1);
        store(dst8, &value, 1);

        ++src8;
        ++dst8;
        --siz;
    }

    return dst;
}
~~~

This function transactionally loads a byte from the source memory, and
transactionally stores it in the destination memory. We could optimize
the implementation by loading and storing memory words instead of bytes,
but the fundamental problem is that the intermediate value exists in the
first place.

To get around this limitation we have to add privatization to our transaction
manager.

#### Privatizing Memory

In these blog posts, we usually start with the implementation of a new
feature and then work our way *outwards* to the interface. This time
let's start with the interface and look at the implementation afterwards.

~~~ c
privatize(uintptr_t addr, size_t siz, bool load, bool, store);
~~~

This looks similar to the `load()` and `store()` function we already have,
doesn't it? The `privatize()` function takes the first address of a memory
buffer and the buffer's length. This is the memory that will be privatized.
The `load` and `store` arguments tell the transaction manager whether we
want to load or store or both.

Using this interface, we can re-implement `memcpy_tx()`.

~~~
void*
memcpy_tx(void* dst, const void* src, size_t siz)
{
    privatize(dst, siz, false, true);
    privatize(src, siz, true, false);

    return memcpy(dst, src, siz);
}
~~~

Here we first privatize `dst` for storing, then privatize `src` for loading.
At this point the transaction owns both buffers. Finally we execute a plain
`memcpy()`.

Not only does the new implementation look much nicer, it has a number of
advantages.

 * Single calls to privatize() have less overhead than repeated calls to
   `load()` or `store()`; simply by the fact that there are less function
   invocations.
 * We got rid of the intermediate value and instead copy directly from
   `src` to `dst`.
 * We use the system's `memcpy()` operation. C compilers or standard
   libraries often replace memory and string functions by highly optimized
   implementations or even single assembly instructions. Using the native
   implementation of `memcpy()` allows these optimization to take place.

#### Implementing Memory Privatization

Let's take a look at the implementation of `privatize()`, especially how
we provide direct access to memory. Let's first add a `flags` field back
to the resource structure.

~~~ c
struct resource {
    uintptr_t       base;
    uint8_t         local_value[RESOURCE_NBYTES];
    uint8_t         local_bits;
    uint8_t         flags;
    pthread_t       owner;
    pthread_mutex_t lock;
};
~~~

Loading is the same as it was before. When we privatize for loading the
transaction acquires the memory resource. The `load()` function would return
a copy of the value, but after privatizing we can just read it directly.

Storing is different than before. So far storing worked by storing to a
transaction-local buffer and later copying the content of this buffer to the
shared memory during a commit. We can call this a *write-back scheme*,
because we first keep a local copy and later *write its content back* to the
global memory location.

When privatizing for store operations the transaction also acquires the
memory. The difference to regular `store()` calls is that for privatizations,
we require what we can call a *write-through scheme.* During the transaction's
execution, we directly *write through* to the shared memory.

This means that when the transaction commits, the shared values are already
updated. For a rollback we have to restore the old value. Here's an
implementation of `privatize()`.

~~~ c
void
privatize(uintptr_t addr, size_t siz, bool load, bool store)
{
    while (siz) {

        struct resource* res = acquire_resource(addr & BASE_BITMASK);
        if (!res) {
            tm_restart();
        }

        unsigned long index = addr & RESOURCE_BITMASK;
        unsigned long bits = 1ul << index;

        uint8_t* beg = arraybeg(res->local_value) + index;
        uint8_t* end = arrayend(res->local_value);

        while (siz && (beg < end)) {
            /* If we're about to store, we first have to
             * save the old value for possible rollbacks. */
            if (store && !(res->local_bits & bits) ) {
                *beg = *((uint8_t*)addr);
            }

            bits <<= 1;
            --siz;
            ++addr;
            ++beg;
        }

        if (store) {
            res->flags |= RESOURCE_FLAG_WRITE_THROUGH;
        }
    }
}
~~~

It's very similar to the load and store functions we developed so far.
The main difference is that for store privatizations, we save the shared
value in the transaction-local buffer in the inner while loop. We will
use this buffer for restoring previous values during a rollback.

For each store-privatized resource, we also set `RESOURCE_FLAG_WRITE_THROUGH`,
which modifies the commit and rollback behavior. Commit and rollback is both
implemented in the same function `release_resource()`.

~~~ c
void
release_resource(struct resource* res, bool commit)
{
    pthread_t self = pthread_self();

    pthread_mutex_lock(&res->lock);

    if (res->owner && res->owner == self) {

        if (res->local_bits) {

            /* We have to store if we either commit in write-back
             * mode, or revert in write-through mode.
             */
            bool store_local_bits = commit != !!(res->flags & RESOURCE_FLAG_WRITE_THROUGH);

            if (store_local_bits) {
                unsigned long bit = 1ul;

                uint8_t* mem = (uint8_t*)res->base;
                uint8_t* beg = arraybeg(res->local_value);
                uint8_t* end = arrayend(res->local_value);

                while (beg < end) {
                    if (res->local_bits & bit) {
                        *mem = *beg;
                    }
                    bit <<= 1;
                    ++mem;
                    ++beg;
                }
            }

            res->local_bits = 0;
            res->flags = 0;
        }

        res->owner = 0;
    }

    pthread_mutex_unlock(&res->lock);
}
~~~

Here we modified the 'write condition.' Before, we only used
write-back mode and unconditionally stored all updates. Now we have to
distinguish between write-back and write-through mode. We have to write
the transaction-local buffer to the shared memory, if

 * we commit in write-back mode (stores), or if
 * we revert in write-through mode (privatizations).

With these modifications we've implemented basic memory privatization. It
should be noted that in the current implementation a transaction can
either load and store *or* privatize a memory resource. The real-world
implementation in [picotm][picotm] handles all combinations of operations
automatically.

#### Producer-Consumer Transactions with Privatizations

What's missing is an update to our example program to use the new
functionality. Here's how the producer looked until now with stores.

~~~ c
static void
producer_func(void)
{
    unsigned int seed = 1;

    tm_save int i0 = 0;

    while (true) {

        sleep(1);

        ++i0;
        int i1 = rand_r(&seed);

        printf("Storing i0=%d, i1=%d\n", i0, i1);

        tm_begin

            store_int(&g_i0, i0);
            store_int(&g_i1, i1);

        tm_commit
    }
}
~~~

We replace the invocations of `store_int()` with `privatize()` and
`memcpy()`.

~~~ c
static void
producer_func(void)
{
    unsigned int seed = 1;

    tm_save int i[2] = {0, 0};

    while (true) {

        sleep(1);

        ++i[0];
        i[1] = rand_r(&seed);

        printf("Storing i0=%d, i1=%d\n", i[0], i[1]);

        tm_begin

            privatize((uintptr_t)g_i, sizeof(g_i), false, true);

            memcpy(g_i, (const void*)i, sizeof(g_i));

        tm_commit
    }
}
~~~

The transaction privatizes the globally shared memory in `g_i` with a
store privatization and copies the updated values there.

The consumer looks similar, but copies *out* of the shared memory.

~~~ c
static void
consumer_func(void)
{
    while (true) {

        sleep(1);

        int i[2];

        tm_begin

            privatize((uintptr_t)g_i, sizeof(g_i), true, false);

            memcpy(i, g_i, sizeof(i));

            verify_load(i[0], i[1]);

        tm_commit

        printf("Loaded i0=%d, i1=%d\n", i[0], i[1]);
    }
}
~~~

#### When not to Privatize?

In theory one could replace all loads and stores with privatizations,
although that's usually not a good idea.

Our implementation of privatization requires a transaction-local backup of
the shared memory's value. For workloads with mostly loads and a low number
of conflicts, these copies could add a non-trivial overhead.

For storing, only *one* transaction can own the privatized shared resource.
With workloads with many store operations to the same memory location, this
could create unnecessary contention. For write-back stores, such overhead
can be reduced by limiting the time stored memory is owned to each
transaction's commit phase.

#### Summary

With this blog post we add memory privatizations to our transaction
manager.

 - Privatizations give transactions direct access to shared resources.
 - This can reduce overhead when wrapping non-transactional code. We used
   `memcpy()` as an example.
 - Memory resources can be privatized by providing a write-through mode
   for store operations.
 - A resource's previous state has to be restored during a rollback
   in write-through mode.
 - Depending on the workload, privatization might have undesirable
   overhead.

You'll find the full source code for this blog post
[on GitHub][source20170601]. If you're interested in a more sophisticated C
transaction manager, take a look at [picotm][picotm].

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

#### Footnotes

[^1]:   Wikipedia describes a slightly different meaning of
        [privatization][wikipedia:privatization] in a slightly different
        context.

[picotm]:                   http://picotm.org/
[posix:memcpy]:             http://pubs.opengroup.org/onlinepubs/9699919799/functions/memcpy.html
[post20170519]:             {% post_url 2017-05-19-writing-the-beginning-and-end-of-a-transaction %}
[post20170526]:             {% post_url 2017-05-26-transactional-access-to-arbitrary-memory-locations %}
[source20170601]:           http://github.com/tdz/blog_examples/tree/master/2017-06-01
[wikipedia:privatization]:  http://en.wikipedia.org/wiki/Privatization_(computer_programming)
