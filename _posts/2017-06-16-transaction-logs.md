---
layout:     post
title:      Transaction Logs
date:       2017-06-16 11:00:00 +0200
tags:       [c, howto, malloc, nosql, posix, stm, transaction, transactional memory, tutorial]
og-image:   share2.png
---

This is series of blog posts to build a transaction manager in C. We
have already implemented support for transactional memory and simple functions
that operate only on memory, such as `memcpy()`. In this installment we're
going to look at logs, transactional logging and transactional memory
allocation.

<!-- excerpt -->

If you missed earlier entries in the series, you might want to go
[back][post:20170505]
[and][post:20170509]
[read][post:20170512]
[the][post:20170519]
[installments][post:20170526]
[so][post:20170601]
[far][post:20170609].

This is the first part of a two-part mini-series on transaction logs. In
this entry we're going to examine the uses for transaction logs and how
they work in theory. In the next entry, we'll implement a log for our
transaction manager *simpletm.*

If you've been following this series over the course of the last weeks,
you might have noticed how we're going from more specific use cases to
more broad scenarios, and how we're going from memory operations to
arbitrary functions.

When we added memory privatization to our transaction manager, we layed the
foundation for calling C functions directly from within a transaction. C
functions often take pointers as their arguments, and memory privatization
is required allow this in a transaction-safe manner.

In this blog post, we'll do the next step towards support for arbitrary
functions. We'll look at transaction logs and functions that require some
form of transactional log.

#### Operations That Have To Be Undone

Let's take a look at the following code fragment.

~~~ c
tm_begin

    void* buf = malloc(size);

    load(...);
    store(...);

tm_commit
~~~

Like many C programs, this transaction allocates memory using `malloc()`. This
memory could be a temporary buffer required for an algorithm executed by the
transaction. The transaction further performs several load and store
operations. It could be a producer transaction that creates data for a
consumer.

Looks good so far, however there's a problem lurking.

Imaging a conflict happens during one of the load or store operations. If the
transaction tries to acquire a memory location that is owned by a concurrent
transaction, the transaction manager will resolve the conflict usually by
rolling back the transaction that came last.

This rollback is a problem because of the memory buffer we allocated. Simply
reverting all load and stores and restarting the transaction will leak the
allocated memory.

To free allocated memory during a rollback, we have to inform the transaction
manager about every allocation we performed during the transaction. This
information is stored in the transaction log.

#### Operations That Have To Be Delayed

Let's further take a look at another example.

~~~ c
void* buf = malloc(size);

tm_begin

    load(...);

    free(buf);

    store(...);

tm_commit
~~~

In this program, memory is allocated outside of the transaction using
`malloc()`. After the transaction started, it loads several memory
locations and performs computations. As part of the transaction the
allocated memory buffer is released using `free()`, and finally some
stores are performed to save the transaction's results. This transaction
could be the consumer to our producer from the previous example.

This example illustrates another problem. Imaging the transaction frees
the buffer and then one of the store operations observes a conflict. The
transaction would have to rollback and restart. The `free()` operation,
however, cannot be rolled back. It has already been performed and the
memory might have already been allocated by another part of the program.

To allow the rollback of a free operation, we have to delay the invocation
of `free()` until the commit happens. This information is also stored in
the transaction log.

#### Undo and Redo

The examples above illustrate the two use case that require a log: some
operations require to be reverted during a rollback, some operations
require to be delayed until commit time.

One can distiguish between undo and redo logs, although implementaions might
not be so strict.

An *undo log* is a log that for each operation stores the old value *before*
that operation. It is useful for reverting operations. In the case of
`malloc()` it would store a pointer to the allocated memory, so that the
transaction manager knows that *before* this invocation of `malloc()`,
the memory had not been allocated.

A *redo log* stores the changes for each operation that affects the state
of the transaction. In the case of `free()` it stores the pointer that
needs to be freed during commit. With this type of log it's possible to
*re-do* the transaction step by step.

We've already seen something like these logs before. Let's for a moment
remember the write-back and write-through semantics that we used for store
operations on transactional memory. For each memory resource, we had a
transaction-local buffer that either stored the original value or the
transaction's modified copy.

In write-back mode the transaction performed all changes in its local buffer,
which was synchronized with the shared global buffer during commit. So the
local buffer acted like a redo log, although it merges all store operations
into one.

A similar thing is true for write-through mode. For a store, the transaction
first copys the shared global value into its local buffer as a backup. Then
it performed modifications directly on the global buffer. During a rollback
the global buffer is reverted to its original state from the content of the
local buffer. So the local buffer acts like an undo log.

#### Implementation Outline

A database usually has an abstract interface to the stored data and the
implemented algorithms. Because access happens though abstract languages
such as SQL, the database's implementation might be able to only go with
an undo or redo log only.

But as mentioned before, an implementation might not be too strict about
distiguishing between undo and redo logs. In our case, we've already seen
that we require both semantics: the undo semantics for `malloc()` and the
redo semantics for `free()`.

Generally speaking, we can say that we require

 - undo semantics for all operations that allocate resources, and
 - redo semantics for all operaionss that release resources.

An operation that allocates a resource can also be a call to `open()` for
opening a file. An operation for releasing a resource can also be a call
to `close()` for closing a file descriptor.

We can simply combine both types of logs into the same data structure. For
each allocating operation we have to

 1. perform the operation, because the result is required immediately by
    the transaction, and
 2. put an entry into the log.

For each releasing operation we have to

 1. put and entry into the log, and
 2. *don't* do anything else; especially no the operation itself.

During a commit we can be sure that the transaction is about to complete. So
we go through the log entries one-by-one and

 - ignore the entry if it refers to an allocating operation. We already
   performed these during the transaction's execution phase.
 - Apply the entry if it is a releasing operation.

During a rollback we go backwards through the log entries one-by-one and

 - undo the entry's effects if it refers to an allocating operation, or
 - ignore the entry if it refers to a releasing operation. We'll certainly
   require the released resource during after the transaction's restart.

As system transactions are about concurrency control and error handling,
each transaction can get by with its own private log. If we have multiple
transactions running concurrently, they won't have to content for access
to a global log. Databases have different use cases and stricter requirements
for their transaction's durability. Therefore they often require logs that
are either shared globally among transactions, or allow for restoring a
global history from the individual transaction's logs. Fortunately none of
this is required in our case.

#### Summary

In this block post, we've investigated the use of logs in our transaction
manager.

 - Resource-allocating operations have to be undone when the transaction
   rolls back.
 - Resource-releasing operations have to be delayed when the transaction
   commits.
 - A transaction log keeps track of all changes performed by a transaction.
 - An undo log stores the state *before* performing an operation.
 - A redo log stores the state change peformed by an operation.
 - Both types of logs can be combined into a single data structure.
 - Each executed operations puts an entry into the transaction's log.
 - During commit the transaction applys all defered operations.
 - During rollback the transaction reverts all performed operations.

So far we only covered some theory and concepts. In the next installment of
this series of blog posts, we will implement a simple transactional log and
implement transactional `malloc()` and `free()` on top of it.

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

[post:20170505]: {% post_url 2017-05-05-implementing-a-simple-transaction-manager-in-c %}
[post:20170509]: {% post_url 2017-05-09-transactional-semantics-in-theory-and-practice %}
[post:20170512]: {% post_url 2017-05-12-transaction-roll-back-and-serializability %}
[post:20170519]: {% post_url 2017-05-19-writing-the-beginning-and-end-of-a-transaction %}
[post:20170526]: {% post_url 2017-05-26-transactional-access-to-arbitrary-memory-locations %}
[post:20170601]: {% post_url 2017-06-01-transactional-memcpy-and-resource-privatization %}
[post:20170609]: {% post_url 2017-06-09-more-on-resource-privatization %}
