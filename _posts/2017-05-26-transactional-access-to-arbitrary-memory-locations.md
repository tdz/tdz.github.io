---
layout:     post
title:      Transactional Access to Arbitrary Memory Locations
date:       2017-05-26 09:00:00 +0200
tags:       [c, howto, nosql, stm, transaction, transactional memory, tutorial]
og-image:   share2.png
---

In this blog post we are going to implement transactional semantics for
arbitrary locations in main memory. So far our *simpletm* transaction manager
can only handle integer values that we store in a dedicated data structure.
At the end of this installment, we'll be able to work with values of arbitrary
types that are located (almost) anywhere in memory.

<!-- excerpt -->

In the [previous][post20170512] [installments][post20170519] we finished our
transaction manager *simpletm* to contain miminal functionality for detecting
and resolving conflicts among transactions, and we separated the transaction
manager from the application logic by creating a generic C interface.

#### Loads and Stores for Integer Resources

For transactional memory access, our example program executes load and store
operations on dedicted data structures of type `struct int_resource`, which
is shown below.

~~~ c
struct int_resource {
    int             value;
    int             local_value;
    unsigned long   flags;
    pthread_t       owner;
    pthread_mutex_t lock;
};
~~~

The shared integer value is stored in the field `value`. Each instance of the
structure is owned by a single transaction that stores in the field
`local_value` and loads from either `value` or `local_value`, depending
whether a local value is present. The `flags` field tells us about this.
During the owning transaction's commit the local value is copied to the actual
value, on rollback the local value is discarded.

The corresponding load/store interface for an application looks like this.

~~~ c
void
load_int(struct int_resource* res, int* value);

void
store_int(struct int_resource* res, int value);
~~~

It's specific to integer values and the application has to know about the
`struct int_resource` data structure. Instead we should prefer an interface
that can store a value of any type at any location in memory.

#### Working With Arbitrary Types

To support arbitrary types, let us rework `struct int_resource` a bit. We
call this new structure `struct resource`.

~~~ c
#define RESOURCE_BITSHIFT   (3)
#define RESOURCE_NBYTES     (1ul << RESOURCE_BITSHIFT)

/**
 * An integer value with an associated owner.
 */
struct resource {
    uintptr_t       base;
    uint8_t         local_value[RESOURCE_NBYTES];
    uint16_t        local_bits;
    pthread_t       owner;
    pthread_mutex_t lock;
};
~~~

First of all, we remove the field `value`. Instead the `base` field tells us
where in memory the resource is located. The value in `base` is the address
of the first byte of a transactional value.

The field `local_value` is now a fixed-size byte array. It still stores
the local value, but it's not any longer type specific. In our case the array
is 8 byte in size, so we can operate on values up to 8 bytes. Further below we
will see how to arrange the resources such that values of arbitrary size can
be supported.

Finally we replaced the `flags` field by `local_bits`. The latter is a bit
field, where each bit signals the present of a transaction-local value in
`local_value`.  So if bit 0 is set `local_value[0]` contains a local value,
if bit 1 is set `local_value[1]` contains a local value, and so on.

#### Working With Arbitrary Memory Locations

Our `local_value` stores 8 bytes and `base` could be any address. If we align
`base` such that it always contains an address at an 8-byte boundary, its
lowest three bytes will always be zero. For a memory address, these bits
contain the index into `local_value`.

Here are the bits of a 32-bit memory address with their meaning for access
to `struct resource`.

```
    | 31 .. 13  12 11 10  9  8  7  6  5  4  3 |  2  1  0 |
    |                  Base                   |   Index  |
```

For mapping a memory address to a byte in a resource structure, first we
have to make sure that the base values of address and resource are the same,
then use the address' lowest three bits as index into `local_value`.

So far we've had two values of type `struct int_resource`. If we had a single
`resource` structure in our transaction manager, we could load and store 8
consecutive bytes for the transactions. To load and store more memory, we
have to add more instances; let's say 1024.

The code fragment below creates an array of 1024 resource structures.

~~~ c
#define NRESOURCES_BITSHIFT (10)
#define NRESOURCES          (1ul << NRESOURCES_BITSHIFT)

struct resource g_resource[NRESOURCES];
~~~

We could assign a base address to each array element and search in the
array whenever we want to map an address to a resource with corresponding
base address. While this would work, there's an easy way to compute the
correct element directly.

Storing the position of element of the array requires exactly 10 bit. These
10 bit are located in the memory address directly before the index; or in
other words, they are the next higher bits above the index.

```
    | 31 .. 13 | 12 11 10  9  8  7  6  5  4  3 |  2  1  0 |
    |          |             Element           |          |
    |                   Base                   |   Index  |
```

For mapping a memory address to a byte in a resource structure, we can
extract the element position (bits 3 to 12), get the element from the
element array we declared, make sure that the base values of address and
resource are the same, and then use the address' lowest three bits as index
into `local_value`.

This look-up scheme is very fast and ensures that resources are consecutive
in memory as well as the bytes of each resource. It is therefore possible to
operate on values that are larger than 8 bytes. For example, if we load or
store a value of 16 bytes, we first operate on the resource structure covering
the value's first bytes, then operate on the resource structure covering the
next bytes, and so on.

#### Loading and Storing

Before we go into the details of the load and store functions, let us look
at the new interfaces. Because we don't use `int` directly any more, the
new interfaces take memory addresses and buffers.

~~~ c
void
load(uintptr_t addr, void* buf, size_t siz);

void
store(uintptr_t addr, const void* buf, size_t siz)
~~~

The value `addr` is the address of the shared value. The application logic
is not required to know about the resource any more. The transaction manager
maintains these internally. The values `buf` and `siz` give a
transaction-local buffer and the number of bytes in the buffer.

The complexity is now in the implementation of the load and store function.
Here is `store()`.

~~~ c
void
store(uintptr_t addr, const void* buf, size_t siz)
{
    const uint8_t* mem = (const uint8_t*)buf;

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
            *beg = *mem;
            res->local_bits |= bits;

            bits <<= 1;
            --siz;
            ++addr;
            ++mem;
            ++beg;
        }
    }
}
~~~

`store()` transactionally stores all bytes in `buf` to the resources that cover
`addr` and successiv memory locations. The function call to
`acquire_resource()` acquires the resource for a specific base address and
returns the resource's data structure. If the resource cannot be acquired,
the transaction is in conflict, `NULL` is returned and the transaction aborts.

Once we have the resource, we start filling its `local_value` field with the
contents of `buf`. For each byte we copy, we set the corresponding bit in
`local_bits`. We do this until we reach the end of either `buf` or
`local_value`. If we reached the end of `local_value` and there are still
bytes left in `buf` we continue the outer while loop with the next resource.

The load function is identical, except for the actual copy operation.

~~~ c
void
load(uintptr_t addr, void* buf, size_t siz)
{
    uint8_t* mem = (uint8_t*)buf;

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
            if (res->local_bits & bits) {
                *mem = *beg;
            } else {
                *mem = *((uint8_t*)addr);
            }

            bits <<= 1;
            --siz;
            ++addr;
            ++mem;
            ++beg;
        }
    }
}
~~~

Here we walk over the local values of the involved resources and copy *into*
`buf`. In the inner while loop, we check if there's a local value and either
copy this or the globally shared value.

Our current setup can support 8192 bytes of transactional memory. If this
limit is reached or if multiple memory locations map to the same resource,
the transaction manager aborts the process. This would be detected by
`acquire_resource()`. As a workaround to this limit the size of the resource
array can be doubled until all variables are covered. A real-world
implementation could also replace the array by a sparse look-up tree. That's
how it's done in [picotm][picotm].

For convenience, we can keep the `load_int()` and `store_int()` functions
around. They are now mere wrappers around `load()` and `store()`.

~~~ c
int
load_int(const int* addr)
{
    int value;
    load((uintptr_t)addr, &value, sizeof(value));
    return value;
}

void
store_int(int* addr, int value)
{
    store((uintptr_t)addr, &value, sizeof(value));
}
~~~

With helpers as simple as these, there can be plenty of type-specific
functions that load and store `float`, `long`, `complex double`, or any
other type.

#### Converting the Application

The old producer code looked like this.

~~~ c
struct int_resource g_int_resource[2];

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

            store_int(g_int_resource + 0, i0);
            store_int(g_int_resource + 1, i1);

        tm_commit
    }
}
~~~

This translates into the following code.

~~~ c
static int g_i0;
static int g_i1;

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

The global values of type `struct int_resource` are gone. Instead we work
directly on integer types.

The consumer side looks similar.

~~~ c
static void
consumer_func(void)
{
    while (true) {

        sleep(1);

        int i0, i1;

        tm_begin

            i1 = load_int(&g_i1);
            i0 = load_int(&g_i0);

            verify_load(i0, i1);

        tm_commit

        printf("Loaded i0=%d, i1=%d\n", i0, i1);
    }
}
~~~

As in the [previous post][post20170519], we added a certain complexity to
the transaction manager, but the application logic got a lot simpler.
And just as before, we had to implement the transaction manager exactly
once. Further application-specific changes are not required.

#### Summary

In this blog post, we removed the dedicated resource data structures from
the interface of *simpletm.* The new interface loads and stores at arbitrary
memory locations and all resource management is done internally.

 - We can load and store arbitrary types.
 - We can protect concurrent access to arbitrary memory locations.
 - Applications don't have to know about dedicated data structures for
   resource management. The interface for transactional memory access
   uses memory addresses of the involved variables.
 - We can further optimize transactional memory access without interfering
   with application logic.

You'll find the full source code for this blog post
[on GitHub][source20170526]. If you're interested in a more sophisticated C
transaction manager, take a look at [picotm][picotm].

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

[picotm]:           http://picotm.org/
[post20170512]:     {% post_url 2017-05-12-transaction-roll-back-and-serializability %}
[post20170519]:     {% post_url 2017-05-19-writing-the-beginning-and-end-of-a-transaction %}
[source20170526]:   http://github.com/tdz/blog_examples/tree/master/2017-05-26
