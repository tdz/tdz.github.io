---
layout:     post
title:      Implementing Fault-Tolerant Software With Transactions
date:       2017-07-21 11:00:00 +0200
tags:       [c, howto, fault-tolerance, fail-safe, nosql, posix, transaction, tutorial]
og-image:   share2.png
---

This is a series of blog posts to build a transaction manager in C. Last time,
we looked at fault tolerance, and how to build fail-safe transactional
software. In this installment, we're going to implement support for error
detection and recovery in our transaction manager *simpletm.* As usual, the
complete example code for this blog post
[is available on GitHub][source:20170721].

<!-- excerpt -->

If you missed earlier entries in the series, you
[might][post:20170505]
[want][post:20170509]
[to][post:20170512]
[go][post:20170519]
[back][post:20170526]
[and][post:20170601]
[read][post:20170609]
[the][post:20170616]
[installments][post:20170623]
[so][post:20170630]
[far][post:20170707]. You might especially want to read the
[previous installment][post:20170714], which in detail lays the foundation
for transactional error handling.

#### A Brief Recap

Here's a very brief recap of the [previous blog post][post:20170714]. In this
installment, we had the following example program written in C.

~~~ c
enum result {
    failure,
    success
};

enum result
func()
{
    if (call_1() == failure) {
        goto err_call_1;
    }
    if (call_2() == failure) {
        goto err_call_2;
    }
    if (call_3() == failure) {
        goto err_call_3;
    }

    return success;

err_call_3:
    revert_call_2();
err_call_2:
    revert_call_1();
err_call_1:
    return failure;
}

void
repair_func()
{
    /* repair errors of func() */
}

int
main(int argc, char* argv[])
{
try_func:
    if (func() == failure) {
        repair_func()
        goto try_func;
    }

    return EXIT_SUCCESS;
}
~~~

The function `main()` calls `func()` until `func()` signals success.
In `func()` itself, there are 3 calls to the functions named `call_1()`,
`call_2()` and `call_3()`. If all invocations succeed, `func()` signals
success; if either invocation fails, `func()` reverts the already completed
calls and signals failure. In case of a failure, `main()` tries to repair
the error by calling `repair_func()`. After completing the repair, `main()`
restarts `func()`.

The revert-repair-and-retry scheme is a typical pattern in C software. It
resembles the rollback-and-retry pattern found in transactions. If we add
the repair functionality to the transaction manager, we can write the code
as a transaction.

~~~ c
void
repair_func()
{
    /* repair errors of func() */
}

int
main(int argc, char* argv[])
{
    tm_begin

        /* execution phase */

        call_1_tx();
        call_2_tx();
        call_3_tx();

    tm_commit

        /* recovery phase */

        repair_func();
        tm_restart();

    tm_end

    return EXIT_SUCCESS;
}
~~~

The functions `call_1_tx()`, `call_2_tx()` and `call_3_tx()` are now invoked
during the transaction's *execution phase.* They internally test for errors
and roll back the transaction if they detect one. This functionality is
implemented by the transaction manager, so there's nothing that the
application programmer has to add.

After an error has been detected and the transaction manager performed the
rollback, it restarts the transaction in the *recovery phase*. During
recovery, the application can try to repair the error and then restart the
transaction's execution phase.

You can already see how much cleaner the transactional implementation is.
Everything that can be automated is handled by the transaction manager's
framework. The only code that has to be written by the application programmer
is the recovery phase, as it's specific to the application and use case.

#### Additional Interfaces for Recovery

For the implementation, we're going to modify our standard example of
producer and a consumer transactions. Let's start with the transaction
manager's public interfaces.

So far, our transactions have been contained by `tm_begin` and `tm_commit`

~~~ c
#define tm_begin                            \
    _tm_begin(setjmp(_tm_get_tx()->env));   \
    {

#define tm_commit                           \
        _tm_commit();                       \
    }
~~~

When used in application code and expanded by the C preprocessor, they
form a block that looks this.

~~~ c
_tm_begin(setjmp(_tm_get_tx()->env));
{

    /* execution phase */


    _tm_commit();
}
~~~

The code for our new recovery phase is located between `tm_commit` and the
new `tm_end` macro. To run either the execution phase or the recovery, we
implement the `tm_begin`-`tm_commit`-`tm_end` combo as if-else branch.

~~~ c
#define tm_begin                                \
    if (_tm_begin(setjmp(_tm_get_tx()->env)))   \
    {

#define tm_commit                           \
        _tm_commit();                       \
    } else {

#define tm_end  \
    }
~~~

Expanded by the C preprocessor, the code looks like this.

~~~ c
if (_tm_begin(setjmp(_tm_get_tx()->env)))
{
    /* execution phase */

    _tm_commit();
} else {

    /* recovery phase */

}
~~~

Whether the transaction enters execution or recovery depends on the value
returned by `_tm_begin()`. And this value is controlled by the transaction
manager. If we start the transaction for the first time or rolled back because
of a conflict, the returned value is *true* and we enter the execution phase.
After we rolled back because of an error, the returned value is *false* and we
enter the recovery phase.

At the end of the recovery phase, the transaction defaults to ending without
retrying execution. To restart we also need a new public interface. It's
called `tm_restart()` and invokes the transaction's execution phase.

~~~ c
void
tm_restart(void);
~~~

#### Error Detection

Our transaction manager contains the function `malloc_tx()`, a transactional
implementation of [`malloc()`][posix:malloc]. Here's the implementation.

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
    save_errno();

    void* ptr = malloc(size);

    append_to_log(NULL, undo_malloc_tx, (uintptr_t)ptr);

    return ptr;
}
~~~

Since `malloc()` fails if no memory is available for allocation, `malloc_tx()`
should detect the error and invoke recovery.

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
    save_errno();

    void* ptr = malloc(size);
    if (!ptr) {
        tm_recover(errno); /* does not return */
    }

    append_to_log(NULL, undo_malloc_tx, (uintptr_t)ptr);

    return ptr;
}
~~~

This implementation of `malloc_tx()` tests the returned pointer and
invokes `tm_recover()` if no memory could be allocated. To understand
`tm_recover()`, let's look at the restart-and-recovery machinery as a
whole.

~~~ c
bool
_tm_begin(int value)
{
    return value != 2;
}

static void
rollback_tx(struct _tm_tx* tx, int value)
{
    release_resources(arraybeg(g_resource),
                      arrayend(g_resource), false);

    /* Revert logged operations */
    undo_log(tx->log, tx->log + tx->log_length);
    tx->log_length = 0;

    /* Restore errno */
    if (tx->errno_saved) {
        errno = tx->errno_value;
        tx->errno_saved = false;
    }

    /* Jump to the beginning of the transaction */
    longjmp(tx->env, value);
}

void
tm_restart()
{
    rollback_tx(_tm_get_tx(), 1);
}

void
tm_recover(int errno_code)
{
    struct _tm_tx* tx = _tm_get_tx();

    tx->recovery_errno_code = errno_code;

    rollback_tx(tx, 2);
}
~~~

The roll back is performed by the internal function `rollback_tx()`. Its
implementation is mostly uninteresting here, but it's important to note that
the second parameter `value` contains the value that is passed to `_tm_begin()`.

If the transaction manager detects a conflict it rolls back the transaction
with a call to `tm_restart()`. This function is also the one to call at the
end of the recovery phase. It uses a value of *1,* which means something like
*regular restart.*

If a transactional function detects an error it invokes `tm_recover()`. This
function saves the errno code and restarts the transaction with a value of
*2.* This value signals the request for error recovery.

An invocation of `_tm_begin()` can now receive three possible values. A value
of *0* signals the initial execution of the transaction. This is the default
value returned by `setjmp()`. A value of *1* signals a restart into the
execution phase and a value of *2* signals a restart into the recovery phase.
`_tm_begin()` tests the value and returns *true* or *false* accordingly.

in addition to the public interfaces, we also add the function
`tm_recovery_errno()`. It returns the stored errno code, so that the
recovery phase can query information about the failure.

~~~ c
int
tm_recovery_errno(void)
{
    struct _tm_tx* tx = _tm_get_tx();

    return tx->recovery_errno_code;
}
~~~

#### A Producer-Consumer Example

In previous blog posts we used an example with a producer transaction and
a consumer transaction. The producer looked something like this.

~~~ c
static void
producer_func(void)
{
    unsigned int seed = 1;

    tm_save int i[2] = {0, 0};

    while (true) {

        sleep(1);

        ++i[0];
        i[1] = rand_r(&seed);

        printf("Storing i0=%d, i1=%d\n", i[0], i[1]);

        tm_begin

            int* buf = NULL;
            load((uintptr_t)&g_i, &buf, sizeof(g_i));

            if (!buf) {

                buf = malloc_tx(2 * sizeof(*buf));
                buf[0] = i[0];
                buf[1] = i[1];

                store((uintptr_t)&g_i, &buf, sizeof(g_i));
            }

        tm_commit
    }
}
~~~

It produces a pseudo-random value and hands it over to the consumer. The
memory is allocated dynamically using `malloc_tx()`.

Let's extend this example with the new functionality. We can add error
recovery by appending only 3 lines of code to the transaction.

~~~ c
static void
recover_from_errno(int errno_code)
{
    fprintf(stderr, "Recovering from error %d (%s)\n", errno_code,
            strerror(errno_code));
}

static void
producer_func(void)
{
    unsigned int seed = 1;

    tm_save int i[2] = {0, 0};

    while (true) {

        sleep(1);

        ++i[0];
        i[1] = rand_r(&seed);

        printf("Storing i0=%d, i1=%d\n", i[0], i[1]);

        tm_begin

            int* buf = NULL;
            load((uintptr_t)&g_i, &buf, sizeof(g_i));

            if (!buf) {

                buf = malloc_tx(2 * sizeof(*buf));
                buf[0] = i[0];
                buf[1] = i[1];

                store((uintptr_t)&g_i, &buf, sizeof(g_i));
            }

        tm_commit
            recover_from_errno(tm_recovery_errno());
            tm_restart();
        tm_end
    }
}
~~~

The new function `recover_from_errno()` is specific to the application. In
a real-world scenario it could try to free some memory, for example by
flushing caches or running a garbage collector. In the example, it simply
prints a message.

To test the code, we can instrument the implementation of `malloc_tx()` to
fail spuriously, just as if the application was low on memory.

~~~ c
static void*
malloc_with_low_mem(size_t size)
{
    /* simulate spurious allocation failures */
    clock_t cputime = clock();
    if (!(cputime % 3)) {
        errno = ENOMEM;
        return NULL;
    }

    return malloc(size);
}

void*
malloc_tx(size_t size)
{
    save_errno();

    void* ptr = malloc_with_low_mem(size);
    if (!ptr) {
        tm_recover(errno); /* does not return */
    }

    append_to_log(NULL, undo_malloc_tx, (uintptr_t)ptr);

    return ptr;
}
~~~

The full example code [is available on GitHub][source:20170721]. If you
run it, it should output something like this to the terminal.

```
Storing i0=1, i1=476707713
Loaded i0=0, i1=0
Storing i0=2, i1=1186278907
Loaded i0=1, i1=476707713
Storing i0=3, i1=505671508
Loaded i0=2, i1=1186278907
Storing i0=4, i1=2137716191
Loaded i0=3, i1=505671508
Storing i0=5, i1=936145377
Loaded i0=4, i1=2137716191
Storing i0=6, i1=1215825599
Loaded i0=5, i1=936145377
Recovering from error 12 (Cannot allocate memory)
Recovering from error 12 (Cannot allocate memory)
Loaded i0=6, i1=1215825599
Storing i0=7, i1=589265238
Recovering from error 12 (Cannot allocate memory)
Loaded i0=7, i1=589265238
```

This is the output of the values stored and loaded by the producer and
consumer transactions. Occationally the producer's memory allocation failed
and triggered the recovery phase.

The final three lines illustrate this nicely. The producer transaction intended
to store value no. 7, but had to recover. The error message tells that memory
allocation failed (as expected). After completing recovery the producer
restarted execution and successfully stored the 7th value. The consumer
transaction then loaded the value correctly, as signalled by the final line of
the output.

#### Summary

In this blog post, we've added support for error detection and recovery
to our *simpletm* transaction manager.

 - Error detection and rollback is provided by the transaction manager's
   framework.
 - Only the error recovery is application-specific and has to be provided
   by the application developer.
 - Transactional functions, such as `malloc_tx`, detect errors and invoke
   recovery.
 - The transaction either restarts into the execution phase or the
   recovery phase.
 - The code of the recovery phase is located between `tm_commit` and
   the new token `tm_end`.

As usual, the complete example code for this blog post [is available on
GitHub][source:20170721].

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

[picotm]:                   http://picotm.org/
[posix:malloc]:             http://pubs.opengroup.org/onlinepubs/9699919799/functions/malloc.html
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
[source:20170721]:          http://github.com/tdz/blog_examples/tree/master/2017-07-21
