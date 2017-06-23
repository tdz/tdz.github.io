---
layout:     post
title:      Transactional malloc() and free()
date:       2017-06-23 12:00:00 +0200
tags:       [c, free, howto, malloc, nosql, posix, stm, transaction, transactional memory, tutorial]
og-image:   share2.png
---

This is a series of blog posts to build a transaction manager in C. In
this installment we're going to implement support for
[`malloc()`][posix:malloc] and [`free()`][posix:free]. With these functions
we can allocate and free blocks of memory, and automatically garbage-collect
allocated blocks if an error or conflict occures during the transaction.

<!-- excerpt -->

If you missed earlier entries in the series, you might want to go
[back][post:20170505]
[and][post:20170509]
[read][post:20170512]
[the][post:20170519]
[installments][post:20170526]
[so][post:20170601]
[far][post:20170609].
You should especially read [last week's entry][post:20170616], which
introduces transaction logs and gives some information about use cases and
scenarios.

#### The Problems of malloc() and free()

One of last week's example transactions looked like this.

~~~ c
tm_begin

    void* buf = malloc(size);

    load(...);
    store(...);

tm_commit
~~~

There's an invocation to `malloc()` near the beginning of the transaction,
and several `load()` and `store()` operations later in the transaction. If
the transaction has to roll back and restart, the malloc'ed memory buffer
would leak. We therefore need a way of freeing the memory during the rollback.

Another example illustrated the problem with `free()`.

~~~ c
void* buf = malloc(size);

tm_begin

    load(...);

    free(buf);

    store(...);

tm_commit
~~~

Here we free a memory buffer that was allocated outside the transaction. If
the transaction has to roll back afterwards, there's no way to un-do the
effects of the `free()` operation. Hence, we'd work on invalid data after a
restart. We can only release memory if we know that the transaction commits.

The solution that we saw in the previous blog post was to keep a log of
operations that either require rollback, or have to be delayed until commit.

#### Extending the Transaction Manager

To extend our transaction manager with a log, we first have to know how
the log entry has to look like. We require a function to apply an operation
during commit, un-do an operation during rollback, and we need some
additional user data for each operation.

So the log entry might look like this.

~~~ c
struct _tm_log_entry {
    void        (*apply)(uintptr_t data);
    void        (*undo)(uintptr_t data);
    uintptr_t    data;
};
~~~

We add an array of these structures to our transaction's internal data
structure.

~~~ c
struct _tm_tx {
    jmp_buf env;

    unsigned long        log_length;
    struct _tm_log_entry log[256];
};
~~~

The log's initial length at the start of the transaction is 0. Using a
static array limits our number of log entries to a certain maximum,
256 in this case, but that's OK for an example. The log in [picotm][picotm]
can grow until it reaches system limits.

How do we append to this log? We have to add a new public interface to the
transaction manager.

~~~ c
void
append_to_log(void (*apply)(uintptr_t),
              void (*undo)(uintptr_t), uintptr_t data)
{
    struct _tm_tx* tx = _tm_get_tx();

    assert(tx->log_length < arraylen(tx->log));

    struct _tm_log_entry* entry = tx->log + tx->log_length;

    entry->apply = apply;
    entry->undo  = undo;
    entry->data  = data;

    ++tx->log_length;
}
~~~

The implementation of `append_to_log()` is fairly trivial. It acquires the
next available element of the log array and fills it with the arguments it
got from the caller. These are the `apply()` and `undo()` functions, and
some user data.

Finally what we need is some extra code to execute these `apply()` and
`undo()` functions. Here's the apply code.

~~~ c
static void
apply_log(struct _tm_log_entry* beg, const struct _tm_log_entry* end)
{
    while (beg < end) {
        if (beg->apply) {
            beg->apply(beg->data);
        }
        ++beg;
    }
}

void
_tm_commit()
{
    release_resources(arraybeg(g_resource),
                      arrayend(g_resource), true);

    struct _tm_tx* tx = _tm_get_tx();

    /* Perform logged operations */
    apply_log(tx->log, tx->log + tx->log_length);
    tx->log_length = 0;
}
~~~

The function `apply_log()` is called by the transaction manager's main commit
function(). It walks over the entries in the log and applys them one by one.
Afterwards the log length is set to 0 again.

The undo code is similar.

~~~ c
static void
undo_log(struct _tm_log_entry* beg, const struct _tm_log_entry* end)
{
    while (end > beg) {
        --end;
        if (end->undo) {
            end->undo(end->data);
        }
    }
}

void
tm_restart()
{
    release_resources(arraybeg(g_resource),
                      arrayend(g_resource), false);

    struct _tm_tx* tx = _tm_get_tx();

    /* Revert logged operations */
    undo_log(tx->log, tx->log + tx->log_length);
    tx->log_length = 0;

    /* Jump to the beginning of the transaction */
    longjmp(tx->env, 1);
}
~~~

The function `undo_log()` walks backwards over the log and un-does each
log entry's effects. It's invoked from the transaction manager's main restart
function. Afterwards the log length is set to 0 again. That's all fairly
trivial, isn't it?

There's one small caveat that we have to watch out for. It's important
that we first apply all operations on memory resources and then apply the
logged operations. Otherwise it could happen that a transaction first frees
a block of memory and then performs store operations on the just free'd
memory. The operation would be undefined.

#### Transactional malloc()

Here's the implementation of our transactional malloc function `malloc_tx()`.

~~~ c
static void
undo_malloc_tx(uintptr_t data)
{
    void* ptr = (void*)data;
    free(ptr);
}

void*
malloc_tx(size_t size)
{
    void* ptr = malloc(size);

    append_to_log(NULL, undo_malloc_tx, (uintptr_t)ptr);

    return ptr;
}
~~~

A call to `malloc_tx()` allocates memory using standard `malloc()`. It
then puts an entry into the log. That entry contains a pointer to the
un-do function and a pointer to the allocated memory. The undo function
`undo_malloc_tx()` simply frees the allocated memory buffer.

Using `malloc_tx()` we can rewrite our first example transaction.

~~~ c
tm_begin

    void* buf = malloc_tx(size);

    load(...);
    store(...);

tm_commit
~~~

Any roll-back of this transaction will un-do the effects of `malloc_tx()`
by freeing the allocated memory.

#### Transactional free()

The transactional free function `free_tx()` looks like this.

~~~ c
static void
apply_free_tx(uintptr_t data)
{
    void* ptr = (void*)data;
    free(ptr);
}

void
free_tx(void* ptr)
{
    append_to_log(apply_free_tx, NULL, (uintptr_t)ptr);
}
~~~

It puts an entry into the log, containing a pointer to the apply function
`apply_free_tx()` and a pointer to the buffer that is to be free'd. The
apply function later executes the free operation during commit.

And our second example using `free_tx()` looks like this.

~~~ c
void* buf = malloc(size);

tm_begin

    load(...);

    free_tx(buf);

    store(...);

tm_commit
~~~

This transaction can roll back at any time. The memory buffer would
only be free'd during commit.

#### Freeing Buffers That Are In Use

There's one possible scenario that we haven't covered yet. What happens
if the free'd buffer is in use by a concurrent transaction?

Let's say we free a buffer in one transaction's commit, but another
transaction concurrently performs a store operation on the buffer's memory.
The buffer is gone so the store operation should observe a conflict. The
current implementation wouldn't detect this.

This problem can be solved fairly easy with memory privatizations. In
`free_tx()` we write-privatize the buffer. This will show up any conflict
between the freeing transaction and any transaction that uses the buffer
concurrently.

~~~ c
void
free_tx(void* ptr)
{
    size_t siz = malloc_usable_size(ptr);

    privatize((uintptr_t)ptr, siz, false, true);

    append_to_log(apply_free_tx, NULL, (uintptr_t)ptr);
}
~~~

The function [`malloc_usable_size()`][man7:malloc_usable_size] is provided
by the GNU C Library. For a given allocated buffer, it returns the size of
the allocation. Most allocators provide an interface like this, but it's
unfortunately not standardized.

#### Summary

In this block post, we've implemented support for transactional `malloc()`
and `free()`.

 - Allocated memory has to be released during rollbacks.
 - Freeing memory cannot be undone, so this operation has to be delayed
   until commit.
 - Invoked operations add themselves to the transaction log.
 - The transaction manager uses the log to un-do `malloc()` operations.
 - The transaction manager uses the log to apply `free()` operations.
 - Conflicts between a freeing transaction and users of the free'd buffer
   can be detected by write-privatizing the buffer before performing the
   free.

As usually, you can find the full source code for this blog post
[on GitHub][source:20170623]. If you're interested in a more sophisticated C
transaction manager, take a look at [picotm][picotm].

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

[man7:malloc_usable_size]:  http://man7.org/linux/man-pages/man3/malloc_usable_size.3.html
[picotm]:                   http://picotm.org/
[posix:free]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/free.html
[posix:malloc]:             http://pubs.opengroup.org/onlinepubs/9699919799/functions/malloc.html
[post:20170505]:            {% post_url 2017-05-05-implementing-a-simple-transaction-manager-in-c %}
[post:20170509]:            {% post_url 2017-05-09-transactional-semantics-in-theory-and-practice %}
[post:20170512]:            {% post_url 2017-05-12-transaction-roll-back-and-serializability %}
[post:20170519]:            {% post_url 2017-05-19-writing-the-beginning-and-end-of-a-transaction %}
[post:20170526]:            {% post_url 2017-05-26-transactional-access-to-arbitrary-memory-locations %}
[post:20170601]:            {% post_url 2017-06-01-transactional-memcpy-and-resource-privatization %}
[post:20170609]:            {% post_url 2017-06-09-more-on-resource-privatization %}
[post:20170616]:            {% post_url 2017-06-16-transaction-logs %}
[source:20170623]:          http://github.com/tdz/blog_examples/tree/master/2017-06-23
