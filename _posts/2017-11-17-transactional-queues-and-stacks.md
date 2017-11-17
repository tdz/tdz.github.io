---
layout:     post
title:      Transactional Queues and Stacks
date:       2017-11-17 12:00:00 +0100
tags:       [data structures, picotm, queues, stacks, stm, transactions]
og-image:   share2.png
---

In [last week's blog post][post:20171107], we've examined transactional
linked lists, a new feature coming in [picotm][picotm] 0.8. In this
blog post, we are going to talk about transactional [queues][github:pr162] and
[stacks][github:pr163]. Like lists, both data structures will be available in
picotm at the end of November.

<!-- excerpt -->

#### A Brief Recap of Transactional Lists

Like traditional linked lists, a transactional linked list is a data
structure for storing data elements. In contrast to the traditional
list, a transactional list can be shared by multiple threads
safely and handles internal errors automatically. As the name suggests,
programs use transactional lists from within a transaction, where they
provide transactional [ACID][wikipedia:acid] properties for all their
operation.

Here's an example program that uses a transactional list to store
values of type `unsigned long`. There's a full explanation of this code
in the [previous blog post][post:20171107],[^1] so let's just go through
the most important points. If you are familiar with the interface of
C++'s Standard Template Library, you'll find many similarities.

~~~ c
    struct ulong_item {
        struct txlist_entry list_entry;

        unsigned long value;
    };

    struct ulong_item item = ULONG_ITEM_INITIALIZER(0);

    struct txlist_state list_state = TXLIST_STATE_INITIALIZER(list_state);
~~~

The variable `item` is a list entry that holds a value of type
`unsigned long`. The value is initialized to `0` by the initializer macro.
The variable `list_state` is the global state of a transactional linked
list. It stores all entries for this list. All actual list access is
performed from within a transaction.

~~~ c
    // Inserting transaction
    //

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

        txlist_push_back_tx(list, &item->list_entry);

        // ... more transactional code ...

    picotm_commit
    picotm_end

    // ... more non-transactional code ...
~~~

The example's first transaction inserts the entry `item` at the end of
the transactional list. It first acquires an actual list from the list
state by calling `txlist_of_state_tx()`. The returned `txlist` structure
`list` provides the interface for operating on the list state, and it
also holds the transaction-local state of the list. With the transactional
list, the first transaction appends the entry to the list state by calling
`txlist_push_back_tx()`. If other transactions access the list state
concurrently, the list's implementation detects and resolves all resulting
conflicts automatically. When the transction commits, the list entry becomes
part of the shared list state.

~~~ c
    // Iterating transaction
    //

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

        struct txlist_entry* beg = txlist_begin_tx(list);
        struct txlist_entry* end = txlist_end_tx(list);

        while (beg != end) {

            struct ulong_item* item = ulong_item_of_entry(beg);

            // ... do something with local item ...

            beg = txlist_entry_next_tx(beg);
        }

        // ... more transactional code ...

    picotm_commit
    picotm_end

    // ... more non-transactional code ...
~~~

The example's second transaction iterates over the elements of a
transactional list. Like the first transaction, it acquires a transactional
list from the list state. It then gets the list's beginning and end entries,
which it uses to traverse the whole list from the first to the final entry.
If other transactions try to modify the list state concurrently, the list
detects and resolves any resulting conflicts automatically.

~~~ c
    // Removing transaction
    //

    picotm_begin

        struct txlist* list = txlist_of_state_tx(&list_state);

        // The list state already contains the entry.
        txlist_erase_tx(list, &item->list_entry);

        // ... more transactional code ...

    picotm_commit
    picotm_end
~~~

Finally, the example's third transaction removes our list entry. It
acquires a list from the list state and calls `txlist_erase_tx()` to
remove the entry. As before, conflicts are detected and resolved
automatically.

#### Transactional Queues

The list is a very generic data structure that serves a wide range of use
cases. If all we want is to insert entries at one end and later retrieve
them in the same order at the other end, we can use a transactional queue
instead. The semantics of a queue allows for more concurrency and can thus
improve performance in some situation. But first, an example.

~~~ c
    struct ulong_item {
        struct txqueue_entry queue_entry;

        unsigned long value;
    };

    struct ulong_item item = ULONG_ITEM_INITIALIZER(0);

    struct txqueue_state queue_state = TXQUEUE_STATE_INITIALIZER(queue_state);

    // Inserting transaction
    //

    picotm_begin

        struct txqueue* queue = txqueue_of_state_tx(&queue_state);

        txqueue_push_tx(queue, &item->queue_entry);

        // ... more transactional code ...

    picotm_commit
    picotm_end

    // ... more non-transactional code ...
~~~

As with lists, there are queue entries and shared queue states. In the
example's first transaction, we acquire a queue for the declared
queue-state variable. And as with lists, the queue is the transactional
interface for modifying the shared queue state. With a call to
`txqueue_push_tx()`, we insert an entry into the queue. When the
transaction commits, the entry becomes part of the shared queue state. As
lists, queues detect and resolve conflicts with concurrent transactions
automatically.

~~~ c
    // Removing transaction
    //

    picotm_begin

        struct txqueue* queue = txqueue_of_state_tx(&queue_state);

        struct txqueue_entry* item = txqueue_front_tx(queue);
        txqueue_pop_tx(queue);

        // ... do something with item ...

        // ... more transactional code ...

    picotm_commit
    picotm_end
~~~

The second transaction in the example removes the entry from the queue.
It reads the next queue entry by calling `txqueue_front_tx()`. This is the
entry we pushed into the queue in the first transaction. Then the second
transaction removes the entry with a call to `txqueue_pop_tx()`. After
the pop operation, the entry itself is still valid, so the transaction can
use it's stored value for further computation. After the successful commit
of the second transaction, the entry is not part of the shared queue
state any longer.

In contrast to lists, entries in a queue cannot be iterated over. Only the
first or final entry can be retrieved. Furthermore, entries can only be
inserted at one end and removed at the other end. These semantics allow for
more concurrency an some situations.

~~~ c
    picotm_begin

        struct txqueue* queue = txqueue_of_state_tx(&queue_state);

        txqueue_push_tx(queue, &item->queue_entry);

        // ... more transactional code ...

    picotm_commit
    picotm_end
~~~

In this example we push an item into the queue. This is a local operation
and because queues don't allow for iteration, there's no reason to integrate
it into the global state just yet. An optimzation applied by picotm is to
keep the push operation in the transaction-local state and only integrate
it into the shared state during a commit. The transaction does not acquire
a reader lock for the queue until it actually commits. Hence concurrent
transactions can retrieve or even remove entries on the queue without being
affected by our uncommitted push.

#### Transactional Stacks

Like queues, stacks have certain properties we can use for reducing conflicts
and increasing concurrency among transactions. But let's look at some code
first. Here's the queue example, modified for stacks.

~~~ c
    struct ulong_item {
        struct txstack_entry stack_entry;

        unsigned long value;
    };

    struct ulong_item item = ULONG_ITEM_INITIALIZER(0);

    struct txstack_state stack_state = TXSTACK_STATE_INITIALIZER(stack_state);

    // Inserting transaction
    //

    picotm_begin

        struct txstack* stack = txstack_of_state_tx(&stack_state);

        txstack_push_tx(stack, &item->stack_entry);

        // ... more transactional code ...

    picotm_commit
    picotm_end

    // ... more non-transactional code ...

    // Removing transaction
    //

    picotm_begin

        struct txstack* stack = txstack_of_state_tx(&stack_state);

        struct txstack_entry* item = txstack_top_tx(stack);
        txstack_pop_tx(stack);

        // ... do something with item ...

        // ... more transactional code ...

    picotm_commit
    picotm_end
~~~

Again we have entries and shared stack state. The stack is acquired with
`txstack_of_stack_tx()` within each transaction. The first transaction in
the example pushes an entry onto the stack, the second transaction retrieves
the stack's top-most entry and removes it. The interfaces are very similar
to those of transactional lists and queues. Of course, each interface detects
and resolves conflicts among transactions automatically.

Similar to queues, the stack allows us to reduce transaction conflicts
and increase concurrency.

~~~ c
    picotm_begin

        struct txstack* stack = txstack_of_state_tx(&stack_state);

        txstack_push_tx(stack, &item->stack_entry);

        // ... more transactional code ...

        txstack_pop_tx(stack);

    picotm_commit
    picotm_end
~~~

In this example, the transaction pushes an entry onto the stack. As with
queues, this operation is entirely local at this point. There's no reason
to acquire a writer lock on the stack state before the transaction actually
commits the change. Concurrent transactions can read the stack state
meanwhile.

Even more concurrency can be achieved within the pop operation. Like a push,
a pop operation modifies the stack's top. In the example above, we push
an entry onto the transaction's local stack top. When we later pop the
entry, we again operate on the local stack top. So unless we pop an entry
from the shared stack state, we don't require any writer locks. If a
transaction performs a number of push operations and afterwards an equal
number of pop operations, effectively no change to the shared state
was performed. In this case, the transaction can commit without acquiring
a lock on the stack state at all.

#### Summary

In this blog post, we've looked at transactional queues and stacks.

 - Like transactional lists, transactional queues and stacks provide
   concurrency control and error detection to implement the ACID
   properties.
 - The semantics of queues and stack can be used to reduce conflicts
   and increase concurrency among transactions.

If you like this blog post, please subscribe to the RSS feed, follow on
Twitter or share on social networks.

#### Footnotes

[^1]:  If you are curious what this is all about and what transactions
       can do for your
       [software][post:20170505],
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


[github:pr162]:             http://github.com/picotm/picotm/pull/162
[github:pr163]:             http://github.com/picotm/picotm/pull/163
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
[wikipedia:acid]:           http://en.wikipedia.org/wiki/ACID
