---
layout:     post
title:      Transactional Multisets
date:       2017-11-24 12:00:00 +0100
tags:       [data structures, multisets, picotm, stm, transactions]
og-image:   share2.png
---

Besides [lists][post:20171107], [queues and stacks][post:20171117]
[picotm][picotm] 0.8 will feature a forth transactional data structures:
[multisets][github:pr167]. A multiset is a sorted set that can contain
the same entry multiple times. In this blog post, we'll go through the
interface and how to use multisets from within transactions.

<!-- excerpt -->

The `struct txmultiset` data structure represents a transactional multiset
(i.e., a set that can hold the same value multiple times). As all the other
transactional data structures it implements concurrency control and error
deteciton in order to provide [ACID][wikipedia:acid] properties for the
multiset.[^1] The multiset interface resembles C++'s Standard Template
Library. If you know C++ multisets, you'll find many similarities.

#### Multiset Entries

To create a multiset, we first need multiset entries. These are represented
by `struct txmultiset_entry`. It works the same way as for lists, queues
and stacks. We can add an instance of this data structure to any data
value to turn it into a multiset entry. Here's an example for multisets
of values of type `unsigned long`.

~~~ c
    struct ulong_item {
        struct txmultiset_entry multiset_entry;

        unsigned long value;
    };

    void
    ulong_item_init(struct ulong_item* item, unsigned long value)
    {
        txmultiset_entry_init(&item->multiset_entry);
        item->value = value;
    }

    struct ulong_item item;

    ulong_item_init(&item, 0);
~~~

This code initializes the multiset entry using `txmultiset_entry_init()`. The
macro `TXMULTISET_ENTRY_INITIALIZER` initializes static or stack-allocated
multiset entries. The example below illustrates this.

~~~ c
    struct ulong_item {
        struct txmultiset_entry multiset_entry;

        unsigned long value;
    };

    #define ULONG_ITEM_INITIALIZER(_value)  \
    {                                       \
        TXMULTISET_ENTRY_INITIALIZER,       \
        (_value)                            \
    }

    struct ulong_item item = ULONG_ITEM_INITIALIZER(0);
~~~

When both, macro and function initialization, is possible, the macro
form is prefered. Multiset entries are uninitialized with
`txmultiset_entry_uninit()`.

~~~ c
    void
    ulong_item_uninit(struct ulong_item* item)
    {
        txmultiset_entry_uninit(&item->multiset_entry);
    }

    ulong_item_uninit(&item);
~~~

#### Multiset States

To store the multiset entries, we need a non-transactional multiset state,
which represents the shared state of a multiset. It's implemented by
`struct txmultiset_state`. We can define and initialize a multiset state
as illustrated in the example below.

~~~ c
    struct ulong_item*
    ulong_item_of_entry(struct txmultiset_entry* entry)
    {
        return picotm_containerof(entry, struct ulong_item, multiset_entry);
    }

    const void*
    ulong_item_key(struct ulong_item* item)
    {
        return &item->value;
    }

    int
    ulong_compare(const unsigned long* lhs, const unsigned long* rhs)
    {
        return (*rhs < *lhs) - (*lhs < *rhs);
    }

    const void*
    key_cb(struct txmultiset_entry* entry)
    {
        return ulong_item_key(ulong_item_of_entry(entry));
    }

    int
    compare_cb(const void* lhs, const void* rhs)
    {
        return ulong_compare(lhs, rhs);;
    }

    struct txmultiset_state multiset_state;

    txmultiset_state_init(&multiset_state, key_cb, compare_cb);
~~~

The example shows the first major difference to the other data structures.
The initializer function takes the multiset state and two additional
call-back functions, which are required for sorting the multiset's
entries. The `key_cb()` call-back function returns a pointer to the key
for a given entry. In the example above, it's the value itself. The
`compare_cb()` call-back function compares two keys. It returns a value
less than, equal to, or greater than zero if the left-hand-side value is
less than, equal to, or greater than the right-hand-side value. Entries in
a multiset are sorted by their key in ascending order. These call-back
functions provide the key and the compare operation.

The code uses the initializer function `txmultiset_state_init()`. For
static or stack-allocated multiset states, there's the initializer macro
`TXMULTISET_STATE_INITIALIZER()`.

~~~ c
    struct txmultiset_state multiset_state = TXMULTISET_STATE_INITIALIZER(multiset_state, key_cb, compare_cb);
~~~

When both forms are possible, the initializer macro is prefered.

Multiset-state clean-up is performed by `txmultiset_state_uninit()`. The
multiset state may not contain entries when the clean-up happens. This
means that entries have to be removed from within a transaction.

For many uses this requirement just imposes unnecessary overhead. A call
to `txmultiset_state_clear_and_uninit_entries()` provies a non-transactional
way of clearing the multiset from its entries. It erases each element from
the multiset and calls a clean-up function on it. The example below shows
the clean-up code for a multiset of `struct ulong_item` entries.

~~~ c
    void
    ulong_item_uninit_cb(struct txmultiset_entry* entry, void* data)
    {
        ulong_item_uninit(ulong_item_of_entry(entry));

        // call free() if item was malloc()'ed
    }

    txmultiset_state_clear_and_uninit_entries(&multiset_state, ulong_item_uninit_cb, NULL);
    txmultiset_state_uninit(&multiset_state);
~~~

#### Acquiring a Multiset

At this point we have a multiset state and multiset entries to add to the state.
So far all code was non-transactional. Actual multiset access and manipulation
is performed by transactional code.

To perform multiset operations within a transaction, we first need a multiset
data structure for our transaction. It's represented by `struct txmultiset`.
A call to `txmultiset_of_state_tx()` returns an instance.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txmultiset* multiset = txmultiset_of_state_tx(&multiset_state);

    picotm_commit
    picotm_end
~~~

Calling `txmultiset_of_state_tx()` multiple times for the same multiset
state *within the same transaction* returns the same multiset. Multisets
are undefined after their transaction committed or aborted, or within
other, concurrent transactions.

#### Inserting Entries

With the multiset, we can now insert entries into the multiset state using
`txmultiset_insert_tx()`. The example below illustrates this.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txmultiset* multiset = txmultiset_of_state_tx(&multiset_state);

        txmultiset_insert_tx(multiset, &item->multiset_entry);

        // more transactional code

    picotm_commit
    picotm_end
~~~

After this transaction committed, the ulong data item `item` will be
the in `multiset_state`. If the transactions has to abort after
the call to `txmultiset_insert_tx()`, the transaction framework will
automatically remove the entry during the rollback; thus restoring the
original state.

The inserted entry will be sorted in ascending order into the multiset,
using the key the compare call-back functions. The insert function compares
the key of the new entry to the keys of existing entries to determines the
new entry's position. The set is implemented as a tree, so inserting requires
a compare operation with only a small fraction of existing items.

#### Removing Entries

To remove an entry from the multiset, we can call `txmultiset_erase_tx()`, as
illustated in the example below.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txmultiset* multiset = txmultiset_of_state_tx(&multiset_state);

        // The multiset state already contains the entry.
        txmultiset_erase_tx(multiset, &item->multiset_entry);

        // more transactional code

    picotm_commit
    picotm_end
~~~

Removing an entry keeps the entries sorted. The multiset's implementation
re-arranges the entries automatically with very little overhead. As usual,
all errors are detected and handled by the transaction framework. The
benefits of transactional code show when we move entries between multisets.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txmultiset* src_multiset = txmultiset_of_state_tx(&src_multiset_state);
        struct txmultiset* dst_multiset = txmultiset_of_state_tx(&dst_multiset_state);

        // The multiset state already contains the entry.
        txmultiset_erase_tx(src_multiset, &item->multiset_entry);

        txmultiset_insert_tx(dst_multiset, &item->multiset_entry);

        // more transactional code

    picotm_commit
    picotm_end
~~~

In this example, we removed an entry from a source multiset and inserted it
into a destination multiset. The transaction framework automatically isolates
these operations from concurrent transactions until the transaction commits.
So concurrent transactions see the entry in *either* the source multiset
*or* the destination multiset, but never in both. If the transaction has to
roll back after the insert operation, the transaction framework
automatically removes the multiset entry from the destination multiset and
returns it to its old position in the source multiset.

#### Multiset Size and Emptieness

A call to `txmultiset_size_tx()` returns the number of multiset entries, a
call to `txmultiset_empty_tx()` returns `true` if a multiset is empty. The
former function might have linear complexity, the later function always has
constant complexity. It's therefore better to use `txmultiset_empty_tx()`
if it's only relevant whether there are entries.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txmultiset* multiset = txmultiset_of_state_tx(&multiset_state);

        bool is_empty = txmultiset_empty_tx(multiset);

        size_t size = txmultiset_size_tx(multiset);

        // more transactional code

    picotm_commit
    picotm_end
~~~

#### Iterating over the Multiset's Entries

We can iterate over the entries of a multiset. The first entry of the
multiset is returned by `txmultiset_begin_tx()`. The terminator entry
*after* the final entry is returned by `txmultiset_end_tx()`. The
terminator entry is not a real entry and should not be dereferenced. Calls
to `txmultiset_entry_next_tx()` and `txmultiset_entry_prev_tx()` return an
entry's successor or predecessor. Here's an example that iterates over a
multiset of values of type `unsigned long` and sums up the individual
values.

~~~ c
    // init and ulong code here

    unsigned long g_sum; // global variable containg sum of multiset entries

    picotm_begin

        struct txmultiset* multiset = txmultiset_of_state_tx(&multiset_state);

        unsigned long sum = 0;

        struct txmultiset_entry* beg = txmultiset_begin_tx(multiset);
        struct txmultiset_entry* end = txmultiset_end_tx(multiset);

        while (beg != end) {

            struct ulong_item* item = ulong_item_of_entry(beg);

            sum += item->value;

            beg = txmultiset_entry_next_tx(beg);
        }

        // more transactional code

        // Picotm's modules collaborate! The value stored in
        // `sum` is exported from the transaction state using
        // `store_ulong_tx()` from the Transactional Memory
        // module.

        store_ulong_tx(&g_sum, sum);

    picotm_commit
    picotm_end
~~~

#### Searching Entries and Ranges

Being a sorted data structure, multisets offer efficient search operations.
A call to `txmultiset_find_tx()` looks-up an entry by a key. A multiset
can contain multiple entries with the same key. In this case
`txmultiset_find_tx()` returns one of them. The exact entry can vary
among calls.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txmultiset* multiset = txmultiset_of_state_tx(&multiset_state);

        unsigned long key = 0;

        struct txmultiset_entry* entry = txmultiset_size_tx(multiset, &key);

        // more transactional code

    picotm_commit
    picotm_end
~~~

The beginning and end of a range of entries with the same key is be
obtained by `txmultiset_lower_bound_tx()` and
`txmultiset_upper_bound_tx()`. The former returns the first entry with the
specified key; the latter returns the entry after the final entry with the
specified key. By iterating over the range, all entries with the key can
be obtained.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txmultiset* multiset = txmultiset_of_state_tx(&multiset_state);

        unsigned int key_count = 0;

        unsigned long key = 0;

        struct txmultiset_entry* beg = txmultiset_lower_bound_tx(multiset, &key);
        struct txmultiset_entry* end = txmultiset_upper_bound_tx(multiset, &key);

        while (beg != end) {

            ++key_count;

            beg = txmultiset_entry_next_tx(beg);
        }

        // ... more transactional code ...

    picotm_commit
    picotm_end
~~~

The above example counts the number of entries for the given key. The
function `txmultiset_count_tx()` performs this operation more efficiently.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txmultiset* multiset = txmultiset_of_state_tx(&multiset_state);

        unsigned long key = 0;

        unsigned int key_count = txmultiset_count_tx(multiset, &key);

    picotm_commit
    picotm_end
~~~

#### Multiset Clean-up

Finally, to clear the whole multiset at once, there's
`txmultiset_clear_tx()`. It's equivalent to a continuous erase operation,
but prefered for its reduced overhead.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txmultiset* multiset = txmultiset_of_state_tx(&multiset_state);

        txmultiset_clear_tx(multiset);

        // more transactional code

    picotm_commit
    picotm_end
~~~

#### Summary

In this blog post, we've looked at transactional multisets.

 - Multisets are sorted sets that can contain the same entry multiple
   times.
 - Like transactional lists, queues and stacks, multisets provide
   concurrency control and error detection to implement the ACID
   properties.
 - A C++-like interface offers access to the multiset's state and allows
   for inserting, removing, iterating and conting the contained entries.
 - Multiset access can be filtered by key, such that only relevant entries
   are processed by a transaction.

If you like this blog post, please subscribe to the RSS feed, follow on
Twitter or share on social networks.

#### Footnotes

[^1]:  If you're interested in the detail of the transaction manager's
       [internals][post:20170505],
       [you][post:20170509]
       [should][post:20170512]
       [go][post:20170519]
       [back][post:20170526]
       [and][post:20170601]
       [read][post:20170609]
       [the][post:20170616]
       [installments][post:20170623]
       [about][post:20170630]
       [the][post:20170707]
       [*simpletm*][post:20170714]
       [toy transaction manager][post:20170721].


[github:pr167]:             http://github.com/picotm/picotm/pull/167
[picotm]:                   http://picotm.org/
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
[post:20171107]:            {% post_url 2017-11-07-transactional-linked-lists %}
[post:20171117]:            {% post_url 2017-11-17-transactional-queues-and-stacks %}
[wikipedia:acid]:           http://en.wikipedia.org/wiki/ACID
