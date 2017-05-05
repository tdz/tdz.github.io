---
layout:     post
title:      Implementing a Simple Transaction Manager in C
date:       2017-05-05 10:05:01 +0200
categories: c howto stm transaction tutorial
---

In this blog post, we're going to write a simple transaction manager
in C. It will load and store integer variables in memory, while handling
locks automatically. The full source code is available on [GitHub][source].

<!-- excerpt -->

A transaction manager provides two important guarantees to each
transaction. These are isolation from the effects of concurrent
transactions, and error handling.[^1] For now we only go for isolation.
Error handling requires a more elaborated software design than what is
possible in a single blog post.

Before writing a transaction manager, the first thing to do is to look at the
involved resources. In our case these are two global integer values.

~~~ c
    int g_i0;
    int g_i1;
~~~

These values are stored in memory. Transaction systems for memory are called
[Software Transactional Memory][stm]. In principle, we could use arbitrary
types of resources or even combine resources of different types in a single
transaction. For simplicity, we stick with memory.

So what do we do with these two values? For our example, we use two threads
in a producer-consumer scenario. The producer thread stores a value in
each of `g_i0` and `g_i1`, the consumer thread reads them.

An important constraint is that the values in these variables are dependent
on each other: either both change or neither changes. We achieve this by
acquiring a lock while accessing ```g_i0``` or ```g_i1```.


#### A non-transactional Example...

Let's take a look at some, still non-transactional, example code. Our producer
looks like this.

~~~ c
    static int g_i0;
    static int g_i1;

    static pthread_mutex_t g_lock0; /* Lock for g_i0 */
    static pthread_mutex_t g_lock1; /* Lock for g_i1 */

    void
    producer_func(void)
    {
        static int i0;
        int i1;

        /* produce i0 and i1 */
        ++i0;
        i1 = rand();

        print("Storing i0=%d, i1=%d\n", i0, i1);

        pthread_mutex_lock(&g_lock0);
        pthread_mutex_lock(&g_lock1);

        g_i0 = i0;
        g_i1 = i1;

        pthread_mutex_unlock(&g_lock0);
        pthread_mutex_unlock(&g_lock1);
    }
~~~

The producer generates two values `i0` and `i1`. The value of `i0` increases
monotonically. It might be a timestamp. The value of `i1` changes arbitrarily;
maybe it's the output of a hardware sensor. By acquiring locks `g_lock0` and
`g_lock1`, the producer enters a critical section where it updates the
global values.

Our consumer looks like this.

~~~ c
    void
    consumer_func(void)
    {
        int i0;
        int i1;

        pthread_mutex_lock(&g_lock1);
        pthread_mutex_lock(&g_lock0);

        i0 = g_i0;
        i1 = g_i1;

        pthread_mutex_unlock(&g_lock0);
        pthread_mutex_unlock(&g_lock1);

        /* consume i0 and i1 */

        print("Loaded i0=%d, i1=%d\n", i0, i1);
    }
~~~

Again we have two local values `i0` and `i1`. This time they are not
generated, but updated from their global counterparts. Like the producer,
the consumer enters a critical section by acquiring the two locks. It
copies the values and prints them to the terminal.

#### ... with a Problem.

This is all nice and good... except it isn't. You've probably spotted
the problem already. Our producer excecutes

~~~ c
    pthread_mutex_lock(&g_lock0);
    pthread_mutex_lock(&g_lock1);
~~~

while our consumer concurrently executes

~~~ c
    pthread_mutex_lock(&g_lock1);
    pthread_mutex_lock(&g_lock0);
~~~

If they interleave, we get

~~~ c
    pthread_mutex_lock(&g_lock0); /* producer */
    pthread_mutex_lock(&g_lock1); /* consumer */
    pthread_mutex_lock(&g_lock1); /* producer */
    pthread_mutex_lock(&g_lock0); /* consumer */
~~~

This is the classical ABBA deadlock. The only way to resolve it is to
detect it beforehand and have one thread release its lock, so that the
other thread can make progress.

Our transaction manager does this for us!

#### Rewriting as Transactions

In a transactional code, each resource is somehow owned by the
transactions that use it. All this is really not the problem of the
transactions themselves, but the transaction manager that controls the
transactions. Each transaction tells the transaction manager what
resources it requires and which operations it performs on them. The
transaction manager handles the details of resource ownership.

As a first step, let us wrap shared integer resources and information
about their current owner in a data structure. This simplifies the later
steps by a large amount.

~~~ c
    struct int_resource {
        int             value;  /* only accessed by owner */
        pthread_t       owner;  /* protected by lock */
        pthread_mutex_t lock;
    };
~~~

This structure has three fields. The `value` field is the value as before.
The `owner` field is the thread of the transaction that currently owns the
resource. A transaction has to acquire ownership from the
transaction manager before it can access the `value` field. Finally, the
field `lock` protects the owner field.

In the next step, we introduce the functions for acquiring and releasing
ownership of a resource.

~~~ c
    bool
    acquire_int_resource(struct int_resource* res)
    {
        pthread_t self = pthread_self();

        pthread_mutex_lock(&res->lock);

        if (res->owner && res->owner != self) {
            /* Owned by another thread. */
            goto err_has_owner;

        } else if (!res->owner) {
            /* Now owned by us. */
            res->owner = self;
        }

        pthread_mutex_unlock(&res->lock);

        return true;

    err_has_owner:
        pthread_mutex_unlock(&res->lock);
        return false;
    }
~~~

The acquire function `acquire_int_resource()` locks the owner field and checks its
value. If someone else is the owner, it reports failure by returning `false`. This
will later lead to an abort of the transaction. If the resource currently has no
owner, `acquire_int_resource()` sets the transaction's thread as the owner and
reports success by returning `true`. If the transaction already owns the resource,
it also reports success.

~~~ c
    void
    release_int_resource(struct int_resource* res)
    {
        pthread_t self = pthread_self();

        pthread_mutex_lock(&res->lock);

        if (res->owner && res->owner == self) {
            /* Release the resource */
            res->owner = 0;
        }

        pthread_mutex_unlock(&res->lock);
    }
~~~

The release function `release_int_resource()` only has to check if the transaction
is the current owner of the resource. If so, it clears the `owner` field. If not,
it returns silently.

You probably noticed that there's not yet a single integer involved here. We could
move this code into a separate module and re-use it for any kind of resource.

The integer values are introduced now. We do this by wrapping the load and store
operations in two functions `load_int()` and `store_int()`. These functions will do
all the work required for accessing shared integer resources.

~~~ c
    bool
    load_int(struct integer_res* res, int* value)
    {
        bool succ = acquire_int_res(res);
        if (!succ) {
            return false;
        }

        *value = res->value;

        return true;
    }

    bool
    store_int(struct integer_res* res, int value)
    {
        bool succ = acquire_int_res(res);
        if (!succ) {
            return false;
        }

        res->value = value;

        return true;
    }
~~~

Both functions first try to acquire the resource by calling
`acquire_int_res()`. If successful they perform their respective operation
by either loading or storing a value. Like the acquire function, they
return true or false, depending on whether or not the resource could be
acquired in the first place.

#### Transactional Producers and Consumers

Phew! I hope you're still with me.

What we've done so far is to write a producer-consumer scenario using locks,
only to discover that our code is prone to deadlocks.

From their we went towards implementing transaction support. We first wrapped
our resource in a data structure, so that is can be maintained by the
transaction manager, then we created interfaces for accessing and releasing
the resource.

Now you might wonder where that transaction manager actually is. Well, in
some way it's already there. The core of our transaction manager is in the
acquire and release functions and in the resource structure. This code does
provide us with everything we need to ensure isolation among transactions.

The only thing missing are the producer and consumer threads that
use our *wanna-be* transaction manager. Let's take a look at the producer.

~~~ c
    static struct int_resource g_int_resource[2];
~~~

The global values are now stored in the instances of `struct int_resource`
in `g_int_resource`.

Like before the producer creates two values and prints them to the terminal.
Unlike before, there are no locks anymore. In fact, the whole critical
section has been replaced by a while loop.

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

You probably remember that the transaction might not be able to acquire
ownership of a resource. In this case, it has to release its locks and
restart from the beginning. That's how we prevent the ABBA deadlock scenario
that our original code couldn't handle.

The transaction starts with `commit` set to `false`. If any invocation of
`store_int()` returns `false`, it was not able to acquire ownership of the
resource. The transaction jumps to `release` where it releases all its
resources. These resources are now available for being acquired by a
concurrent transaction. Since `commit` is still `false`, the while loop
continues and the transaction starts anew.

If all invocations of `store_int()` succeed, the transaction eventually
sets `commit` to `true`. It will then also release its resources, but now
the while loop breaks. The transaction has committed its results.

The while loop is good for educational purposes, not an optimal software
design, and not what you'd use in a real-world transaction manager. We'll
build something better in a later blog post.

Now that we have a producer, let's also look at the consumer side.

~~~ c
    void
    producer_func(void)
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

Again we replaced the critical section by a while loop. The principles of
aborting and committing the transaction are the same here as in the producer,
just that this transaction calls the `load_int()` function.

If you run this example, it should display something like the output below
on the terminal.

```
    Loaded i0=0, i1=0
    Storing i0=1, i1=1804289383
    Loaded i0=1, i1=1804289383
    Storing i0=2, i1=846930886
    Storing i0=3, i1=1681692777
    Loaded i0=3, i1=1681692777
```

#### Summary

I hope you enjoyed the quick tour through a very simple transaction manager.
While the presented code is simple, it already contains many of the basic
building blocks of a complete implementation.

 - The transaction manager provides isolation and error handling to its
   transactions.
 - Transactions interact with transactional resources by invoking
   transactional interfaces.
 - Non-transactional code is prone to deadlocks, transactional code
   isn't.[^2]
 - Conflicting access is resolved by the transaction manager. This can
   lead to an abort and restart of the involved transactions.

The full source code for this blog post is [available on GitHub][source].

There are more blog posts to come that talk about all kinds of topics around
transactions. If you're interested in a complete transaction manager take a
look at [picotm][picotm].

#### Footnotes

[^1]:   In database context, these guarantees are known as the [ACID][acid]
        properties.

[^2]:   We'll talk about livelocks in a later blog post.

[acid]:     https://en.wikipedia.org/wiki/ACID
[picotm]:   https://picotm.github.io/
[source]:   https://github.com/tdz/blog_examples/tree/master/2017-05-05
[stm]:      https://en.wikipedia.org/wiki/Software_transactional_memory
