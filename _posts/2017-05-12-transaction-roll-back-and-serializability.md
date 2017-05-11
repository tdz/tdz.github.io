---
layout: post
title:  Transaction Roll-Back and Serializability
date:   2017-05-12 09:00:00 +0200
tags:   [c, howto, serializability, stm, transaction, transactional memory, tutorial]
---

In this blog post we'll explore transaction roll-back code and briefly cover
serializability. We'll build upon our
[simple transaction manager][post20170505] and the
[theoretical foundation][post20170509] we've discussed last time. At the end
of this post, we'll have a simple, but correct transaction manager that can
be adapted to arbitrary work loads.

<!-- excerpt -->

#### A Closer Look at *simpletm*

Let's take a closer look at *simpletm,* the transaction manager that we
made before. Specifically let's look at the producer and consumer code.
Here's the producer again.

~~~ c
void
producer_func(void)
{
    static int i0;
    int i1;

    /* produce i0 and i1 */
    ++i0;
    i1 = rand();

    print("Storing i0=%d, i1=%d\n", i0, i1);

    bool commit = false;

    while (!commit) {

        bool succ = store_int(g_int_resource + 0, &i0);
        if (!succ) {
            goto release;
        }
        succ = store_int(g_int_resource + 1, &i1);
        if (!succ) {
            goto release;
        }

        commit = true;

    release:
        release_int_resource(g_int_resource + 0);
        release_int_resource(g_int_resource + 1);
    }
}
~~~

It creates two values, `i0` and `i1`, and stores them in shared resources;
acquiring these resources along the way. If it can not acquire a resource,
the producer releases all its resources and restarts.

Here's the consumer.

~~~ c
void
consumer_func(void)
{
    int i0, i1;

    bool commit = false;

    while (!commit) {

        bool succ = load_int(g_int_resource + 1, &i1);
        if (!succ) {
            goto release;
        }
        succ = load_int(g_int_resource + 0, &i0);
        if (!succ) {
            goto release;
        }

        commit = true;

    release:
        release_int_resource(g_int_resource + 0);
        release_int_resource(g_int_resource + 1);
    }

    /* consume i0 and i1 */

    print("Loaded i0=%d, i1=%d\n", i0, i1);
}
~~~

Similar to the producer, the consumer loads two values from shared resources
and acquires these resources along the way. If it fails to acquire a resource
the consumer releases all its other resources and restarts.

Let's further condense both code blocks to the load, store, and release
functions that operate on shared resources. A transaction won't see conflicts
when accessing its private resources. So for our analysis, it's only
important how transactions interact with shared resources.

~~~ c
void
producer_func(void)
{
    store_int(g_int_resource + 0, &i0);
    store_int(g_int_resource + 1, &i1);
    release_int_resource(g_int_resource + 0);
    release_int_resource(g_int_resource + 1);
}

void
consumer_func(void)
{
    load_int(g_int_resource + 1, &i1);
    load_int(g_int_resource + 0, &i0);
    release_int_resource(g_int_resource + 0);
    release_int_resource(g_int_resource + 1);
}
~~~

Producer and consumer run concurrently, so invocations of the load, store,
and release functions can interleave in any way. Let's reorder the
individual calls... and a problem will appear.

~~~ c
store_int(g_int_resource + 0, &i0);         /* producer acquires resource 0 and performs store */

/* context switch */

load_int(g_int_resource + 1, &i1);          /* consumer acquires resource 1 and performs load */

/* context switch */

store_int(g_int_resource + 1, &i1);         /* producer fails to acquire resource 1 */
release_int_resource(g_int_resource + 0);   /* producer releases resource 0 */

/* context switch */

load_int(g_int_resource + 0, &i0);          /* consumer acquires resource 0 and performs load */

/* THE CONSUMER JUST LOADED THE VALUE PREVIOUSLY STORED BY THE ABORTED PRODUCER! */
~~~

Oh! If we only execute the first store and the first load, and then abort
the producer, the consumer will load the value that has just been stored by
the producer.

Is this actually correct? Before, we could only have guessed, but in the
previous blog post, we established a theoretical foundation and talked about
the ACID properties.

Our property of atomicity required transactions to be either executed
completely, or not at all. In the example at hand we only executed half of
the producer's transaction, namely the first store; therefore violating
the atomicity principle.

Consequently we also violated the isolation property, as we leaked internal
changes of the producer into the global state.

We specified early int the original blog post that we want to update either both
resources or none. So we also violated our established notion of consistency.

Clearly, the current behavior is not correct.

#### Test it for Yourself

Usually you won't see this problem, because the `sleep()` and `printf()`
statements slow down thread execution considerably. If you want to trigger
the bug, take the [original code of the simpletm][source20170505] and
replace `producer_func()` and `consumer_func()` with the code shown below.

~~~ c
static void
producer_func(void)
{
    unsigned int seed = 1;
    int i0 = 0;

    while (true) {

        ++i0;
        int i1 = rand_r(&seed);

        bool commit = false;

        while (!commit) {

            bool succ = store_int(g_int_resource + 0, i0);
            if (!succ) {
                goto release;
            }
            succ = store_int(g_int_resource + 1, i1);
            if (!succ) {
                goto release;
            }

            commit = true;

        release:
            release_int_resource(g_int_resource + 0);
            release_int_resource(g_int_resource + 1);
        }
    }
}

static void
verify_load(int i0, int i1)
{
    unsigned int seed = 1;

    int i;
    int value = 0;

    for (i = 0; i < i0; ++i) {
        value = rand_r(&seed);
    }

    if (value != i1) {
        printf("Incorrect value pair (%d,%d), should be (%d,%d)\n",
               i0, i1, i, value);
    }
}

static void
consumer_func(void)
{
    while (true) {

        int i0, i1;

        bool commit = false;

        while (!commit) {

            bool succ = load_int(g_int_resource + 1, &i1);
            if (!succ) {
                goto release;
            }
            succ = load_int(g_int_resource + 0, &i0);
            if (!succ) {
                goto release;
            }

            verify_load(i0, i1);

            commit = true;

        release:
            release_int_resource(g_int_resource + 0);
            release_int_resource(g_int_resource + 1);
        }
    }
}
~~~

The replacement code has the invocations of `sleep()` and `printf()` removed
and verifies the loaded values. When you run it, you should see plenty of
statements like the ones below on the terminal.

```
Incorrect value pair (17103,1980393415), should be (17103,507428599)
Incorrect value pair (17104,507428599), should be (17104,1058612930)
Incorrect value pair (17111,1861807294), should be (17111,2062867441)
Incorrect value pair (17112,2062867441), should be (17112,1510167188)
```

This comes from the consumer thread, which failed to verify the correctness
of the loaded values.

Notice that if `i0` is 17103 the consumer expects `i1` to be 507428599, but
this value is only loaded later when `i0` is 17104. This is because the
producer stored 17104 for `i0` and then aborted, keeping `i1` at the old
value. The same happens for 17111 and 17112.

#### Introducing Serializability

Remember from the previous post that the cumulative effects of a set of
transactions shall look as if the transactions had been executed one by
one. This is called serializability.

A specific execution of a set of transactions is called a *history.* For
example, the execution near the beginning of this post, the one that
showed up the problem, is a history.

In our code we have two transactions that we execute concurrently. If we
execute all operations of one transaction strictly before or after all
operations of the other transaction, we have a *serial history.* This
generalizes to histories containing any number of transactions.

A history is *serializable* if the effects of history's committed
transactions are equivalent to an actual serial history. This means that
if we execute and commit all our transactions concurrently, the result must
be the same as if we had executed and committed the transactions one by one.
The exact order of the transactions doesn't matter.

To make a history look like a serial one, we have ensure is that each
transaction always sees the shared state as if no other transaction was
present. If a transaction detects a conflict, it has to revert its own
changes as if they never happened.

#### Fixing the Code

Our transaction manager already detects conflicts among transactions that
access the same resource concurrently. This means that we have already
implemented half of the requirements for serializability. What the
transaction manager is still missing is a way of reverting a transaction's
changes when the transaction aborts.

We implement this by introducing transaction-local state for each resources.

~~~ c
#define RESOURCE_HAS_LOCAL_VALUE    (1ul << 0)

struct int_resource {
    int             value;
    int             local_value;    /* the transaction-local state */
    unsigned long   flags;          /* flag set if local state present. */
    pthread_t       owner;
    pthread_mutex_t lock;
};
~~~

The owner of the resource always stores only to the local state; and it
always loads from the local state if that is present.

~~~ c
static bool
store_int(struct int_resource* res, int value)
{
    bool succ = acquire_int_resource(res);
    if (!succ) {
        return false;
    }

    /* only store to local state */
    res->local_value = value;
    res->flags |= RESOURCE_HAS_LOCAL_VALUE;

    return true;
}

static bool
load_int(struct int_resource* res, int* value)
{
    bool succ = acquire_int_resource(res);
    if (!succ) {
        return false;
    }

    /* return the local state if we have executed a store() before */
    if (res->flags & RESOURCE_HAS_LOCAL_VALUE) {
        *value = res->local_value;
    } else {
        *value = res->value;
    }

    return true;
}
~~~

The local state becomes globally visible if, and only if, (!) the transaction commits.
This happens just before releasing the resource.

~~~ c
static void
release_int_resource(struct int_resource* res, bool commit)
{
    pthread_t self = pthread_self();

    pthread_mutex_lock(&res->lock);

    if (res->owner && res->owner == self) {

        /* if we commit, copy the local state to the global state */
        if (res->flags & RESOURCE_HAS_LOCAL_VALUE) {
            if (commit) {
                res->value = res->local_value;
            }
            res->flags &= ~RESOURCE_HAS_LOCAL_VALUE;
        }

        res->owner = 0;
    }

    pthread_mutex_unlock(&res->lock);
}
~~~

Every time we call `release_int_resource()`, we now have to tell it whether
we're comitting or not.

~~~ c
static void
producer_func(void)
{
    [...]
            commit = true;

        release:
            release_int_resource(g_int_resource + 0, commit);
            release_int_resource(g_int_resource + 1, commit);
        }
    }
}
~~~

If we commit a transaction, its local effects become globally visible. If we
abort, the transaction's effects are thrown away.

These fairly small changes fix the incorrect state that the consumer was
observing. If you run this code, even with `sleep()` and `printf()` removed,
you won't see any warnings about incorrect value pairs.

#### Summary

In this post, we looked at serializability and how to implemenent it in our
transaction manager.

 - A *serial* history is a set of transactions that are executed one by one.
 - We want concurrent executions of a history to have the same result as
   a serial one. This is called *serializability.*
 - We have to revert a transaction's changes when the transaction aborts.
 - We introduced transaction-local state for each resource. The local state
   becomes globally visible *iff* the transaction commits.

As before, the full source code for this blog post is
[available on GitHub][source20170512]. The code should now be complete enough
to be useful for an arbitrary amount of producers and consumers, or threads
that perform both.

In the next installment, we'll replace the while loop in the producer and
consumer functions with something nicer. The result will be a cleaner and
more maintainable code base.

[picotm]:           https://picotm.github.io/
[post20170505]:     {% post_url 2017-05-05-implementing-a-simple-transaction-manager-in-c %}
[post20170509]:     {% post_url 2017-05-09-transactional-semantics-in-theory-and-practice %}
[source20170505]:   https://github.com/tdz/blog_examples/tree/master/2017-05-05
[source20170512]:   https://github.com/tdz/blog_examples/tree/master/2017-05-12
