---
layout:     post
title:      Transactional Linked Lists
date:       2017-11-07 10:00:00 +0200
tags:       [data structures, lists, picotm, stm, transactions]
og-image:   share2.png
---

The November release of [picotm][picotm] will feature
[transactional lists][github:pr159].
These lists can be accessed and modified concurrently by multiple
transactions while respecting the transaction's ACID properties. In this
blog posts, we're going to look at the lists' features and interface.

<!-- excerpt -->

#### The Transaction Interface

Transactional code runs isolated from other transactions in the system. All
concurrency and errors are handled by a transaction manager. If anything
fails the transaction framework rolls the transaction back into the original
state and either retries or invokes error recovery. To the transaction it
looks as if it had the whole system for itself and errors never happen. This
is generally known as the [ACID properties][wikipedia:acid].

We can build this easily with picotm, a transaction manager for C applications
and system software. Without going into details, a picotm transaction looks as
shown in the example below. The transactional code is located between
`picotm_begin` and `picotm_commit`.

~~~~ c
    picotm_begin

        // Transactional code

    picotm_commit

        // Error-recovery code

    picotm_end
~~~~

Earlier in this blog we had a series of articles about constructing a
transaction manager in C. If you want to know how all this works at the
fundamental level, you
[might][post:20170505]
[want][post:20170509]
[to][post:20170512]
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

#### Why Do We Want Lists?

The code in a transactions runs in its own, isolated space. To interact with
the outside world (i.e., the rest of the program) it has to perform
operations on shared data structurs. Such a data structure could be a
double-linked list.

For an example, imagine one transaction generates list entries with
values received over the network from another computer or read from a file.
It enqueues the entries into a globally shared list where they are picked
up by other transactions for further processing of the contained data.

The list has to be transactional to make this work. Therefore it must
automatically handle concurrency among transactions and detect internal
errors. On any error it has to be able to go back into its original state.
In the rest of this blog article well examine how to do this in practice
with the new transactional lists in picotm.

The interfaces are similar to the lists of C++ by design. You'll probably
recogonize some if you have experience with C++' Standard Template Library.

#### List Entries

To create a list, we first need list entries. These are represented by
`struct txlist_entry`. We can add an instance of this data structure to any
data value to turn it into a list entry. Here's an example for lists of
values of type `unsigned long`.

~~~ c
    struct ulong_item {
        struct txlist_entry list_entry;

        unsigned long value;
    };

    void
    ulong_item_init(struct ulong_item* item, unsigned long value)
    {
        txlist_entry_init(&item->list_entry);
        item->value = value;
    }

    struct ulong_item item;

    ulong_item_init(&item, 0);
~~~

This code initializes the list entry using `txlist_entry_init()`. There's
also an initializer macro. `TXLIST_ENTRY_INITIALIZER` initializes static
or stack-allocated list entries. The example below illustrates this.

~~~ c
    struct ulong_item {
        struct txlist_entry list_entry;

        unsigned long value;
    };

    #define ULONG_ITEM_INITIALIZER(_value)  \
    {                                       \
        TXLIST_ENTRY_INITIALIZER,           \
        (_value)                            \
    }

    struct ulong_item item = ULONG_ITEM_INITIALIZER(0);
~~~

When both, macro and function initialization, is possible, the macro
form is prefered. List entries are uninitialized with
`txlist_entry_uninit()`.

~~~ c
    void
    ulong_item_uninit(struct ulong_item* item)
    {
        txlist_entry_uninit(&item->list_entry);
    }

    ulong_item_uninit(&item);
~~~

#### The Shared List State

To store the list entries we need non-transactional list state, which
represents the shared state of a double-linked list. It's implemented
by `struct txlist_state`. We can define and initialize a list state
as illustrated in the example below.

~~~ c
    struct txlist_state list_state;

    txlist_state_init(&list_state);
~~~

This code uses the initializer function `txlist_state_init()`. For
static or stack-allocated list states, there's the initializer macro
`TXLIST_STATE_INITIALIZER()`.

~~~ c
    struct txlist_state list_state = TXLIST_STATE_INITIALIZER(list_state);
~~~

When both forms are possible, the initializer macro is the prefered form.

List-state clean up is performed by `txlist_state_uninit()`. The list
state may not contain entries when the clean-up happens. This means that
entries have to be cleaned up from within a transaction.

For many uses this requirement just imposes unnecessary overhead. A call to
`txlist_state_clear_and_uninit_entries()` provies a non-transactional
way of clearing the list from its entries. It erases each element from
the list and calls a clean-up function on it. The example below shows
the clean-up code for a list of `struct ulong_item` entries.

~~~ c
    struct ulong_item*
    ulong_item_of_entry(struct txlist_entry* entry)
    {
        return picotm_containerof(entry, struct ulong_item, list_entry);
    }

    void
    ulong_item_uninit_cb(struct txlist_entry* entry, void* data)
    {
        ulong_item_uninit(ulong_item_of_entry(entry));

        // call free() if item was malloc()'ed
    }

    txlist_state_clear_and_uninit_entries(&list_state, ulong_item_uninit_cb, NULL);
    txlist_state_uninit(&list_state);
~~~

At this point we have a list state and list entries to add to the state.
So far all code was non-transactional. Actual list access and manipulation
is performed by transactional code.

#### Finally, Lists!

To perform list operations within a transaction, we first need a list
data structure for our transaction. It's represented by `struct txlist`.
A call to `txlist_of_state_tx()` returns an instance.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

    picotm_commit
    picotm_end
~~~

Calling `txlist_of_state_tx()` multiple times for the same list state
*within the same transaction* returns the same list. Lists are undefined
after their transaction committed or aborted, or within other, concurrent
transactions.

#### Adding And Removing List Entries

With the list, we can now append or prepend entries to the list state
using `txlist_push_back_tx()` and `txlist_push_front_tx()`. The example
below illustrates the use of `txlist_push_back_tx()`.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

        txlist_push_back_tx(list, &item->list_entry);

        // more transactional code

    picotm_commit
    picotm_end
~~~

After this transaction committed, the ulong data item `item` will be
the final entry in `list_state`. If the transactions has to abort after
the call to `txlist_push_back_tx()`, the transaction framework will
automatically remove the appended entry during the rollback; thus
restoring the original state.

To remove an entry from the list, we can call `txlist_erase_tx()`,
illustated in the example below.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

        // The list state already contains the entry.
        txlist_erase_tx(list, &item->list_entry);

        // more transactional code

    picotm_commit
    picotm_end
~~~

As usual, all errors are detected and handled by the transaction
framework. The benefits of transactional code show when we move
entries between lists.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txlist* src_list = txlist_of_state_tx(&src_list_state);
        struct txlist* dst_list = txlist_of_state_tx(&dst_list_state);

        // The list state already contains the entry.
        txlist_erase_tx(src_list, &item->list_entry);

        txlist_push_back_tx(dst_list, &item->list_entry);

        // more transactional code

    picotm_commit
    picotm_end
~~~

In this example, we remove an entry from a source list and append it to
a destination list. The transaction framework automatically isolates these
operations from concurrent transactions until the transaction commits. So
concurrent transactions see the entry in *either* the source list *or* the
destination list, but never in both. If the transaction has to roll back
after the append operation, the transaction framework automatically removes
the list entry from the destination list and returns it to its old position
in the source list.

#### List Capacity

A call to `txlist_size_tx()` returns the number of list entries, a call to
`txlist_empty_tx()` returns `true` if a list is empty. The former function
might have linear complexity, the later function always has constant
complexity. It's therefore better to use `txlist_empty_tx()` if it's only
relevant whether there are entries.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

        bool is_empty = txlist_empty_tx(list);

        size_t size = txlist_size_tx(list);

        // more transactional code

    picotm_commit
    picotm_end
~~~

#### Iterating Over List Entries

We can also iterate over the entries of a list. The first entry of the list
is returned by `txlist_begin_tx()`. The terminator entry *after* the final
entry is returned by `txlist_end_tx()`. The terminator entry is not a real
entry and should not be dereferenced. Calls to `txlist_entry_next_tx()` and
`txlist_entry_prev_tx()` return an entry's successor or predecessor. Here's
and example that iterates over a list of values of type `unsigned long` and
sums up the individual values.

~~~ c
    // init and ulong code here

    unsigned long g_sum; // global variable containg sum of list entries

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

        unsigned long sum = 0;

        struct txlist_entry* beg = txlist_begin_tx(list);
        struct txlist_entry* end = txlist_end_tx(list);

        while (beg != end) {

            struct ulong_item* item = ulong_item_of_entry(beg);

            sum += item->value;

            beg = txlist_entry_next_tx(beg);
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

If we have an entry in a list, we can insert another entry *before*
the existing one with `txlist_insert_tx()`. In the example below, we
insert before the terminator entry, which is equivalent to an append
operation.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

        txlist_insert_tx(list, &item->list_entry, txlist_end_tx(list));

        // more transactional code

    picotm_commit
    picotm_end
~~~

To append or prepend entries to a list, it's better use the appropriate
push functions, though. The transaction framework might be able to optimize
concurrency control for each. Also, inserting or removing from a list
can make iteration loops invalid. It's therefore better to separate these
operations cleanly.

#### Clearing The List

Finally, to clear the whole list at once there's `txlist_clear_tx()`.
It's equivalent to a continuous erase operation, but prefered for its
reduced overhead.

~~~ c
    // init and ulong code here

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

        txlist_clear_tx(list);

        // more transactional code

    picotm_commit
    picotm_end
~~~

#### Next Steps

The current transactional lists already provide everything we need. But
of course, there's always room for optimizations.

One possible next step is the introduction of an iterator data type that
abstracts access to list entries. Such an abstraction would allow
for more concurrency when operating on lists, as the order of list entries
would be determined by the iterator and not the actual list order.

With iterators in place, appending and prepending to a list is another
possible area of optimization. Each operation means *modify the entry at
either end of the list,* but neither depends on where the actual end is.
Each transaction can have its own local beginning and end of the list.
Only during a commit would this local state become part of the global list
state.

#### Summary

In this blog post, we've looked at transactional lists and how to use them.

 - Transactional lists provide concurrency control and error detection to
   implement the ACID properties.
 - Non-transactional list state and entries are separate from the
   transactional list.
 - All list access and modification is performed by transactions.
 - A transaction creates a list by acquiring global list state.
 - Transactions can add and remove entries in a list, retrieve the number
   of entries in a list, and iterate over a list's entries.
 - Sophisticated list implementations can optimize concurrency control
   depending on each list operation's semantics.

If you like this blog post, please subscribe to the RSS feed, follow on
Twitter or share on social networks.

[github:pr159]:             http://github.com/picotm/picotm/pull/159
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
[wikipedia:acid]:           http://en.wikipedia.org/wiki/ACID
