---
layout:     post
title:      Types of Transactional Operations
date:       2017-07-07 10:00:00 +0200
tags:       [c, howto, irrevocability, nosql, posix, stm, transaction, transactional memory, tutorial]
og-image:   share2.png
---

This is a series of blog posts to build a transaction manager in C. In this
installment, we're going to take a closer look at the different types of
functions that exist and what we have to do to handle them in a transaction.

We've already implemented support for several interfaces from the C Standard
Library in our transaction manager. We're now going to sort and formalize
this a bit.

<!-- excerpt -->

If you missed earlier entries in the series, you might
[want][post:20170505]
[to][post:20170509]
[go][post:20170512]
[back][post:20170519]
[and][post:20170526]
[read][post:20170601]
[the][post:20170609]
[installments][post:20170616]
[so][post:20170623]
[far][post:20170630].

#### Overview

Initially we only had support for memory operations in our transaction
manager. On top of that, we added memory and string helper functions, such
as `memcpy()`. With the introduction of the transaction log, we have also
added support for `malloc()` and `free()`, which required either reversal
during roll back or delay to commit.

We can distiguish 4 different classes of transactional functions. A
transactional function is either

 - pure,
 - deferable,
 - revocable, or
 - irrevocable.

Sorted roughly by their impact on the transaction system, these classes
differ in the kind of support they require from the transaction manager
and consequently the amount of work they require for an implementation.

#### Pure Functions

*Pure functions* don't have any dependencies or side effects that interfere
with the transaction.

One example on Linux systems is [`pthread_self()`][posix:pthread_self],
which returns the calling thread's id. The function doesn't depend on any
parameters or mutable state. The returned id is constant for each thread.
In a transaction, we can call `pthread_self()` without further considerations.

Another example is [`fmin()`][posix:fmin], a function that returns the
smaller of two values of type `double`. The parameters it receives are the
two values to compare. Again, no pointers or mutable state that this function
depends on. A call to `fmin` doesn't even generate errors. There's a result
for all possible input values, even *NaN.*

A `load()` operation on transactional memory is also pure. While its result
depends on the memory's content, `load()` doesn't change non-transactional
mutable state.

Handling pure functions is easy. We can call them directly from within the
transaction.

#### Deferable Functions

The next class after pure functions are deferable functions. A *deferable
function* does not have an immediate effect on the transaction and can be
defered until commit time.

A common deferable function that we've already seen is [`free()`][posix:free].
Free releases a block of dynamically allocated memory. It accepts a pointer
to the memory block as its only argument. After `free()` returns, the
content of the memory block is invalid and accessing it is an undefined
operation.

If we'd free the memory during the transaction's execution, there'd be no
way of rolling back the transaction later, as free operations cannot be
reverted.

However, there's no need to actually free the memory at this point. Instead
we defer the free operation until commit time. In the case of a roll back
of the transaction, we simply ignore the call to `free()`.

Another example of a deferable function is the `store()` operation we
implemented for transactional memory. Stores in write-back mode buffer
the written data and defer it until commit time.

The main criteria for a deferable function is that it has an effect on
non-transactional mutable state, but the effect is either not immeditely
visible, or can be simulated within the transaction.

#### Revocable Functions

In some way, a revocable function is the opposite of a deferable function.
A *revocable function* is executed during the transaction, but it's effects
are revoked (or reverted) during the transaction's roll back.

In our case, we've already seen [`malloc()`][posix:malloc] as an example of
a revokable function. A call to `malloc()` returns a dynamically allocated
block of memory. This operation has to be executed immediately during the
transaction as the memory might be required during subsequent operations
within the transaction.

To support `malloc()`, the transaction executes the operation immediately
and returns the allocated memory block. If it rolls back the transaction
later, the transaction manager frees the allocated block automatically.

Another example is a [`read()`][posix:read] operation on a file. Whatever
gets read might have an effect on the transaction, so the data returned by
the read operation is immediately required. But reading has no side-effects
on the data itself. It mostly changes the file offset at which the next read
operation takes place. The change to the offset can be revoked by restoring
the old offset during a roll back. So reading from a file is a revocable
operation.

As a third example, again `store()` can act like a revocable function. When
executed in write-through mode, `store()` immediately writes the data to
memory. It keeps a copy of the old data, which is restored during a roll
back.

Revocable functions have an effect on the transaction's execution and
therefore have to be performed immediately. The effects on non-transactional
mutable state can be revoked during a roll back.

#### Irrevocable Functions

Finally we have irrevocable functions. The result of an *irrevocable
function* is immeditely required, but there's no way of revoking it's effects
on non-transactional mutable state.

An example of an irrevocable function is a `read()` operation on a Unix pipe,
which is often used to transfer data from one program to another program.
Similar to a file, invoking `read()` on a pipe returns the next block of
data. Unlike a file, a pipe doesn't provide any means of reverting the effect
of the read operation. Once it has read data from the pipe, the transaction
has no way of putting back the data into a pipe during a roll back. The
effect of calling `read()` on a pipe is therefore not revocable. The function
is thus irrevocable.

The only way to support irrevocable functions is to guarantee that a
transaction is not rolled back after it executed an irrevocable function. This
has a number of consequences for the transactional system as a whole.

 - A transaction has to explicitly request irrevocability from the
   transaction manager. This requires support by the transaction manager.

 - Once it a transaction is irrevocable, it wins any conflict with
   concurrent transactions.

 - At any time, there can only be one irrevocable transaction. Imagine
   there were two irrevocable transactions at the same time. If there's
   a conflict between these two transactions, it would not be possible
   to pick a loser without violating irrevocability.

The easiest way of supporting irrevocability is to run the irrevocable
transaction exclusively. When a transaction requests irrevocability, all
other transactions are stopped, rolled back, and suspended until the
irrevocable transaction committed. So there are either multiple revocable
transactions running, or at most one irrevocable transaction. This is the
same semantics as for reader/writer lock and that's how it's implemented.

A more elaborated implementation would use a lock manager. This component
of the transaction manager observes the state of resource locks, and resolves
conflicts between transactions. When it has to resolve a conflict between the
irrevocable transaction and a revocable transaction, it would always pick the
irrevocable transaction as winner.

To summarize, irrevocable functions have an effect on the transaction's
execution and therefore have to be performed immeditely. The effects on
non-transactional mutable state cannot be revoked during a roll back, so
the transaction may not perform a roll back at all.

#### Special Cases

Aside from the 4 basic types, there are a number of special cases.

First of all, there are functions that are both deferable and recovable. One
such function is [`realloc()`][posix:realloc], which changes the size of an
already allocated block of memory. A call to `realloc()` has to be performed
immediately as the allocated memory has to be returned into the transaction.
This memory has to be released during a roll back. The memory block of the
old size also has to be released, but only at commit time. An implementing
of a transactional `realloc()` is a combination of `malloc()` and `free()`.

Another special case happens when functions change their type depending
on some internal state. We've already seen `store()` and `read()`. These
functions' type depends on parameters and internal state.

Finally, there are operations that simple don't make sense in the context
of a transaction. These usually concern program state as a whole. Calling
functions like [`exit()`][posix:exit], [`exec()`][posix:exec], or
[`abort()`][posix:abort] will end the current program in some way. Calling
them within a transaction would probably lead to an inconsistent state of
the system.

#### Summary

In this blog post, we've discussed the different types of transactional
functions.

 - Pure functions have no effect on non-transactional state.
 - Deferable functions are defered until commit time. Their effects on
   non-transactional state are simulated to the transaction.
 - Revocable functions are executed immediately during the transaction,
   but their effects on non-tranactional state are reverted during a
   roll back.
 - Irrevocable functions are executed immediately, but their effects on
   non-transactional state cannot be reverted. A transaction may not
   have to roll back after executing an irrevocable function.
 - Functions can change their type, depending on the parameters or the
   transaction's state.
 - Some functions have no meaningful semantics in the context of
   transactions.

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

[man7:malloc_usable_size]:  http://man7.org/linux/man-pages/man3/malloc_usable_size.3.html
[picotm]:                   http://picotm.org/
[posix:abort]:              http://pubs.opengroup.org/onlinepubs/9699919799/functions/abort.html
[posix:exec]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/exec.html
[posix:exit]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/exit.html
[posix:fmin]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/fmin.html
[posix:free]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/free.html
[posix:malloc]:             http://pubs.opengroup.org/onlinepubs/9699919799/functions/malloc.html
[posix:pthread_self]:       http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_self.html
[posix:read]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/read.html
[posix:realloc]:            http://pubs.opengroup.org/onlinepubs/9699919799/functions/realloc.html
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
[source:20170630]:          http://github.com/tdz/blog_examples/tree/master/2017-06-30

[tuhs]: http://www.tuhs.org/


[unixv4:errno]:         http://github.com/dspinellis/unix-history-repo/blob/Research-V4-Snapshot-Development/sys/user.h#L39
[unixv4:man_perror]:    http://github.com/dspinellis/unix-history-repo/blob/Research-V4-Snapshot-Development/man/man3/perror.3#L28
[unixv5:errno]:         http://github.com/dspinellis/unix-history-repo/blob/Research-V5-Snapshot-Development/usr/source/s4/perror.c#L1

[freebsd:errno]:        https://github.com/dspinellis/unix-history-repo/commit/4a81bbc3e8796e1f18f1a5e8ba62836ad71025c2#diff-4c402341f93d93b29dc0f6eb539600f5R46
