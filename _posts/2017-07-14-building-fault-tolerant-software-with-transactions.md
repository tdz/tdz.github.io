---
layout:     post
title:      Building Fault-Tolerant Software With Transactions
date:       2017-07-14 11:00:00 +0200
tags:       [c, howto, fault-tolerance, fail-safe, nosql, posix, transaction, tutorial]
og-image:   share2.png
---

This is a series of blog posts to build a transaction manager in C. Having
implemented thread isolation, we will now look at error handling. Our goal
is to automate error handling as much as possible and make building
fault-tolerant software easy.

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
[far][post:20170707].

#### Overview

In the very first installment of the series, I claimed that the
transaction manager provides thread isolation and error handling
to its transactions; with a clear implication that transactional
code would do better than its traditional counter part.  But so
far we've only looked at thread isolation. We have built a basic
implementation of Software Transactional Memory and added support
for a few C functions on top.

In this and the next blog post, we're going to discuss error handling
and automate it as much as possible. This time we cover the theory, and
next time we add an implementation to our *simpletm* transaction
manager. In the end, we'll have infrastructure that allows for easy
creation of fault-tolerant software.

#### The Tradional Approach

A typical pattern of C code looks like this.

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
~~~

The function `func()` invokes 3 other functions named `call_1()`, `call_2()`,
and `call_3()`. If either fails, the execution moves to the end of `func()`
and executes a block of clean-up code to revert the effects of the already
completed functions.

Although hard coded, these goto statements and labels act like an undo log
in a transaction: all the executed operations are un-done and the system
moves back into its previous state.

When `func()` returns `failure`, the calling code could try to repair the
problem and retry the invocation.

~~~ c
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

Here, the `main()` function invokes `func()` and tries to repair any errors
that might happen. Erroneous invocations are re-started after the repair.

This pattern works very well in traditional C programs, but has a few
drawbacks.

 - The error handling interleaves with the application code. The application
   logic, the error detection and the error recovery is all mixed in the
   same location.
 - The error handling is hard coded and cannot be re-used easily. Similar
   code requires similar error handling, which has to be re-written. Any
   changes later on have to be applied to all error-handling code.
 - The code is not well structured. There are quite a few goto statements
   and labels. In more complex programs, it's not always easy to follow
   these jumps.

#### The Transactional Approach

Transactions can help to solve these problems. As with thread isolation,
application logic is separate from the error detection. The former is
given by the application programmer, the latter is entirely implemented
within the transaction framework. Error recovery also is provided by
the application programmer, but separated from the application logic.

To implement error handling, let us first extend our transaction manager's
interface a bit. So far, a transaction is only confined by `tm_begin` and
`tm_commit`. All transactional application logic is listed between these
two statements. This is called *execution phase*. Once the transaction
reached `tm_commit`, it entered the *commit phase.*

We now extend this interface with a new statement called `tm_end`.

~~~ c
tm_begin

    /* execution phase */

tm_commit   /* commit phase */

    /* recovery phase */

tm_end
~~~

`tm_end` follows after `tm_commit` and marks the end of the transaction.
The code between `tm_commit` and `tm_end` is provided by the application
programmer to repair possible errors. We call this the *recovery phase.*
The transaction framework only invokes recovery if it detects an error.

An error-free transaction still looks like before.

 1. Perform `tm_begin`.
 2. Perform the execution phase.
 3. Perform `tm_commit`.
 4. Leave the transaction by executing the next instruction after `tm_end`.

An erroneous transaction instead runs the recovery phase.

 1. Perform `tm_begin`
 2. Perform the execution phase up to the point were the error happens. The
    error is detected within the framework. All transactional operations;
    such as `load()`, `store()`, `malloc_tx()` or `free_tx()`; are provided
    by the transaction manager. Their implementation detects the error
    internally. The transactional application logic does not have to do
    error detection.
 3. Roll-back the transaction log. In traditional code, an equivalent
    operation is performed by the clean-up code at the end of `func()`.
 4. Perform the recovery phase. The recovery code is application specific
    and has to be provided by the application programmer.
 5. After successful recovery, restart the transaction's execution phase.
 6. Perform `tm_commit`.
 7. Leave the transaction by executing the next instruction after `tm_end`.

If done correctly, there is no difference between the result of an error-free
run, and the result of an erroneous run with successful recovery.

Although rather abstract, let's re-write the original example as transactional
code.

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

        call_1_tx();
        call_2_tx();
        call_3_tx();

    tm_commit

        repair_func();
        tm_restart();

    tm_end

    return EXIT_SUCCESS;
}
~~~

We've replaced the function `func()` with a transaction. During the
execution phase, the transaction invokes `call_1_tx()` to `call_3_tx()`
These operations run the respective `call_?()` function and append
them to the transaction log, together with an undo function. We've seen
how this works in detail in the
[blog post on transactional `malloc()`][post:20170623]. Shown below is
a pseudo implementation of `call_1_tx()`.

~~~ c
static void
undo_call_1_tx(uintptr_t data)
{
    revert_call_1();
}

void
call_1_tx()
{
    if (call_1() == success) {
        append_to_log();
        append_to_log(NULL, undo_call_1_tx, 0);
    } else {
        tm_recover(); /* does not return */
    }
}
~~~

The functions `call_1_tx()` to `call_3_tx()` also perform error detection
internally. If either function fails, it instructs the transaction manager
to perform a rollback and invoke the recovery phase.

The recovery phase starts with a call to `repair_func()`. What exactly it
does is dependent on the application; maybe it runs the garbage collector to
free memory, or cleans up files to free disk space. In any case it can at
least send an email to the administrator and shut down the software
gracefully, so even sever errors don't go unnoticed.

After a successful repair, the recovery code invokes `tm_restart()`, which
restarts the transaction's execution phase. It now either succeeds by
committing the transaction or detects another error and re-runs the
recovery phase.

You can see from this abstract example, how we've solved the traditional
code's problems with error handling.

 - We cleanly separated application logic from error handling. For the
   error handling, we further separated error detection (i.e., done within
   the framework) from error recovery (i.e., done during the recovery phase).
 - Error detection is implemented once by each transactional operation. The
   detection code is re-used for each transaction and freely combined with
   other operation's error detection.
 - Error recovery has to be implemented only once by the application
   programmer. The code in `repair_func()` could repair all kinds of errors,
   not just those specific to our example.
 - All goto statements and labels are gone, making the transactional code
   much more readable.

#### Summary

In this blog post, we've discussed the problems of error handling, and looked
at the solution the transaction's provide.

 - Traditional code interleaves application logic, error detection and
   error recovery; making the result hard to read and maintain.
 - Transactional code detects errors within the transaction manager's
   framework.
 - Each transaction contains a specific *recovery phase* that implements
   recovery strategies for specific errors.
 - If the transaction framework detects an error, it rolls back the
   transaction's execution and runs the recovery code.
 - After a successful recovery, the transction can be re-executed.
 - Transactional code cleanly separates application logic, error detection
   and error recovery.
 - The result of an erroneous transaction with successful recovery is the
   same as the result of the same transasction after and an error-free run.

This blog post was fairly high-level and abstract. In the next installment,
we'll add an implementation to our *simpletm* toy transaction manager. All
this (and more) is already provided by [picotm][picotm], the system-level
transaction manger.

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

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
