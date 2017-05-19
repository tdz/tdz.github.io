---
layout:     post
title:      Writing the Beginning and End of a Transaction
date:       2017-05-19 09:00:00 +0200
tags:       [c, howto, longjmp, nosql, setjmp, stm, transaction, transactional memory, tutorial]
og-image:   share2.png
---

[Last time][post20170512] we finished our transaction manager and example
program *simpletm* to contain the minimum functionality for conflict detection
and transaction rollback. In this blog post we are going to separate
application logic and transaction logic from each other. We will move the
latter into a distinct module with a generic C interface for beginning and
committing transactions.

<!-- excerpt -->

#### Towards a Generic Transaction Interface

*Simpletm* has two threads. A producer thread generates pairs of integer
values and stores them in two globally shared variables. The consumer thread
loads the stored values from the shared variables. All load and store
operations are performed transactionally.

We ended the previous blog post with a producer function that looked like the
code shown below.

~~~ c
static void
producer_func(void)
{
    unsigned int seed = 1;

    int i0 = 0;

    while (true) {

        sleep(1);

        ++i0;
        int i1 = rand_r(&seed);

        printf("Storing i0=%d, i1=%d\n", i0, i1);

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
            release_int_resource(g_int_resource + 0, commit);
            release_int_resource(g_int_resource + 1, commit);
        }
    }
}
~~~

While this code works correctly, it's poorly designed and not generally
re-useable in other software.

 - The inner while loop serves as transaction control block. This
   is not obvious. We should prefer clear statements that indicate when we
   begin and commit a transaction.
 - We explicitly test the return value of each call to `store_int()` and
   abort if necessary. This is error prone and obfuscates the appliction
   logic. It would be preferable if `store_int()` could do this automatically
   for us.
 - We maintain the variable `commit` by hand. That is also error prone. If we
   forget to set `commit` to `true`, we end up in an end-less loop. If we
   set it too early (i.e., before all loads and stores have been executed),
   we might commit a conflicting transaction.
 - We track which resources the transaction uses and release them explicitly
   at the end of the transaction. It would be better if this could be done
   once for all transactions.

The consumer has similar problems. If we can move all this behind a
common interface, we can reduce the overall code to the pure application
logic.

#### The Interface

First of all we have to mark the beginning of a transactional block. Once
a transaction has began, there are currently two cases we must handle: either
our transaction commits and leaves the transaction, or the transaction aborts
at some point and returns to the beginning of the transaction block. In both
cases we have to release the transaction's acquired resources.

Let's start with implementing two statements. We call them `tm_begin` and
`tm_commit`.

As the name indicates, `tm_begin` is where the transaction starts. Here's
the code.

~~~ c
void
_tm_begin()
{
    /* Nothing to do */
}

#define tm_begin    \
    _tm_begin();    \
    {
~~~

The function `_tm_begin()` is the place where we can put any set-up code that
we might require. It's currently empty.[^1] The function is not supposed to
be called directly. Instead there's `tm_begin`. It's a macro that calls
`_tm_begin()` and then begins a new C code block. Note that `tm_begin`
intentionally has no round brackets. It's not a function call. It's an
element of code structuring.

The code for `tm_commit` is where we update and release the shared resources
acquired by the transaction.

~~~ c
#define arraylen(_array)    \
    ( sizeof(_array) / sizeof(*(_array)) )

#define arraybeg(_array)    \
    ( _array )

#define arrayend(_array)    \
    ( arraybeg(_array) + arraylen(_array) )

void
release_int_resources(struct int_resource* beg,
                const struct int_resource* end, bool commit)
{
    while (beg < end) {
        release_int_resource(beg, commit);
        ++beg;
    }
}

void
_tm_commit()
{
    release_int_resources(arraybeg(g_int_resource),
                          arrayend(g_int_resources), true);
}

#define tm_commit       \
        _tm_commit();   \
    }
~~~

If `tm_begin` marks the beginning of our transactional code, `tm_commit`
marks the end. It's again a macro. It calls the function `_tm_commit()`
and ends the C code block that was started by `tm_begin`. The intended
side effect of this block is that all variables declared in the
transactional code are transaction-local by default.

The commit operation's implementation is in `_tm_commit()`. This function
releases all shared integer resources. Then we have completed the
transaction. The rest of the shown code is simply helpers.

The primitives shown so far is all we need for beginning and committing
transactions. But what about conflict handling?

#### Non-Local Gotos in C

When a transaction observes a conflict during resource access, we want it
to release the resources it acquired so far and restart from the beginning.

Conflicts are detected within `store_int()`, or `load_int()` for the
consumer transaction. To restart from there we have to implement a non-local
goto that jumps from `store_int()` to where we called `tm_begin`. It's called
*non-local goto* because we're doing a goto operation over function-call
boundaries.

Non-local gotos in C are implemented with the functions
[setjmp()][posix::setjmp] and [longjmp()][posix::longjmp].[^2] A call to
`setjmp()` stores the processors current instruction counter and stack pointer
for this thread. A later call to `longjmp()` restores these values, effectivly
transfering program control back to the point where `setjmp()` had been called.
The state values are stored in a *jump buffer* of type `jmp_buf`.

Here's how the code looks like. First we need a thread-local jump buffer.

~~~ c
struct _tm_tx {
    jmp_buf env;
};

struct _tm_tx*
_tm_get_tx()
{
    /* Thread-local transaction structure */
    static __thread struct _tm_tx t_tm_tx;

    return &t_tm_tx;
}
~~~

We put the jump buffer into `struct _tm_tx`. The function `_tm_get_tx()`
simply returns a thread-local instance of this structure.

To save the thread state in the jump buffer upon entering the transaction,
we have to call `setjmp()` at the right time. Therefore we modify `tm_begin`
as shown below.

~~~ c
void
_tm_begin(int value)
{
    /* Nothing to do */
}

#define tm_begin                            \
    _tm_begin(setjmp(_tm_get_tx()->env));   \
    {
~~~

The call to `setjmp()` is now done at the very beginning. It saves the
instruction counter and stack pointer in the thread-local jump buffer
and returns a value. When we later call `longjmp()` with the saved jump
buffer, the transaction performs the non-local goto and `setjmp()` will
return *again.*

Upon entering the transaction for the first time, `setjmp()` returns 0. At
the time we call `longjmp()` we have the option of passing a different value,
which `setjmp()` will return. Its return value is passed as an argument to
`_tm_begin()`. This way we can distinguish between a new transaction and
a restarted transaction in `_tm_begin()`.

What we finally need is the restart function that is executed by `store_int()`
upon detected conflicts.

~~~ c
void
tm_restart()
{
    release_int_resources(arraybeg(g_int_resource),
                          arrayend(g_int_resource), false);

    /* Jump to the beginning of the transaction */
    longjmp(_tm_get_tx()->env, 1);
}

void
store_int(struct int_resource* res, int value)
{
    bool succ = acquire_int_resource(res);
    if (!succ) {
        tm_restart();
    }

    res->local_value = value;
    res->flags |= RESOURCE_HAS_LOCAL_VALUE;
}
~~~

Like `_tm_commit()` the function `tm_restart()` releases the transaction's
resources, but of course without updating the global values. Instead of
returning, `tm_restart()` invokes `longjmp()` with the saved jump buffer and
an additional value. This transfers program control back to the transaction's
beginning where `setjmp()` saved the jump buffer. This time, `setjmp()`
returns the second argument to `longjmp()`. In our case this is 1.

In `store_int()`, we now call `tm_restart()` upon detected conflicts. Notice
that a call to `store_int()` cannot fail anymore. If it returns it has been
successful, otherwise it automatically restarts the transaction.

#### Putting It All Together

While the details of `setjmp()` and `longjmp()` are certainly more complex
than the while loop, it's all contained in the transaction manager. And we're
effectivly done with this problem. There's hardly any reason to ever change
the code.

The application logic, however, looks *much* clear than before. Let's put
everything together. Here's our new producer.

~~~ c
static void
producer_func(void)
{
    unsigned int seed = 1;

    int i0 = 0;

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

We were able to condense the whole inner while loop into four statements
were each statement is relevant to the application! Gone are the while loop,
the explicit tests for successful stores, the `commit` variable, and the
explicit releases at the end of the transaction.

After applying similar changes to the consumer, the new consumer looks like
this.

~~~ c
static void
consumer_func(void)
{
    while (true) {

        sleep(1);

        int i0, i1;

        tm_begin

            load_int(g_int_resource + 1, &i1);
            load_int(g_int_resource + 0, &i0);

            verify_load(i0, i1);

        tm_commit

        printf("Loaded i0=%d, i1=%d\n", i0, i1);
    }
}
~~~

Again, we were able to remove all the transactional book keeping and reduce
the code to the application logic!

#### Optimizing for Optimizers

There's one final point about non-local gotos that we have to take care of.
Remember that `setjmp()` only saves the processor's instruction counter and
stack pointer. It *does not* save any program variable's value.

Program variables are subject to opimizations by the compiler. They can even
be removed entirely and the value can be hold in a processor register. If we
compile the *simpletm* binary with the gcc flags '-O2 -Wclobbered', we might
get a warning like the one shown below.

```
main.c: In function ‘producer_func’
main.c:33:9: warning: variable ‘i0’ might be clobbered by ‘longjmp’ or ‘vfork’ [-Wclobbered]
```

This means that the compiler's optimizer moves around access to `i0` or
maybe even removes the variable. Such optimizations depend on the function's
structure and code. The information about how the `longjmp()` happens
is simply not available to the optimizer because the call to `longjmp()`
happens in a different function. After a `longjmp()` the value of `i0`
is undefined.

We can prevent this from happening by declaring the `i0` as `volatile`, or
as we call it `tm_save`. The `volatile` keyword prevents the compiler from
optimizing the variable.

~~~ c
#define tm_save volatile

static void
producer_func(void)
{
    unsigned int seed = 1;

    tm_save int i0 = 0;

    [...]
}
~~~

From now on, we will compile transactional code with '-Wclobbered' to make the
compiler warn about this problem, and we will add `tm_save` where necessary.

#### Summary

In this blog post, we created a clean interface that let's us separate our
transaction manager from the application logic and keep both in separate
modules.

 - We begin a transaction with `tm_begin` and commit with `tm_commit`.
 - Both statements are implemented by the transaction manager.
 - A call to `tm_begin` saves the instruction counter and stack pointer using
   `setjmp()`. For aborting we execute `longjmp()` to return to the beginning
   of the transaction.
 - Transaction aborts are performed internally by the transaction manager.
 - Resources are released automatically by the transaction manager during
   commits and aborts. No support from the application is required.
 - The application code is reduced to the actual application logic.

As always, the full source code for this blog post is
[available on GitHub][source20170519]. If you're interested in a more
sophisticated C transaction manager, take a look at [picotm][picotm].

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.
In the next installment, we will probably replace `struct int_resource` with
support for arbitrary memory locations.

#### Footnotes

[^1]:   We will fill `_tm_begin()` with code when we get to error recovery
        in a later blog post.

[^2]:   As word of warning, try to avoid using `setjmp()` and `longjmp()` in
        your software. It's perfect for our transactions, but in general it
        often does more harm than good to program design.

[picotm]:           http://picotm.org/
[posix::longjmp]:   http://pubs.opengroup.org/onlinepubs/9699919799/functions/longjmp.html
[posix::setjmp]:    http://pubs.opengroup.org/onlinepubs/9699919799/functions/setjmp.html
[post20170512]:     {% post_url 2017-05-12-transaction-roll-back-and-serializability %}
[source20170519]:   http://github.com/tdz/blog_examples/tree/master/2017-05-19
