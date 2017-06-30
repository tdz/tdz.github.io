---
layout:     post
title:      Everything About errno; plus Transactions
date:       2017-06-30 10:00:00 +0200
tags:       [c, errno, howto, malloc, nosql, posix, stm, transaction, transactional memory, tutorial]
og-image:   share2.png
---

This is a series of blog posts to build a transaction manager in C. In this
entry, we'r going to take a look at `errno` and how it works behind the
scenes. Afterwards we're going to add support for `errno` to our transaction
manager *simpletm*.

<!-- excerpt -->

If you missed earlier entries in the series, you might want
[to][post:20170505]
[go][post:20170509]
[back][post:20170512]
[and][post:20170519]
[read][post:20170526]
[the][post:20170601]
[installments][post:20170609]
[so][post:20170616]
[far][post:20170623]. Entries are self-contained as much as possible, but
knowning the big picture might by helpful in some cases.

#### Communicating Problems

The C and POSIX standards specify a multitude of ways for reporting errors
to a program.

If you do something critical, such as accessing unmapped memory regions,
you'll get a signal distributed from the operating system; for example
SIGSEGV in the case of unmapped memory. Usually these critical signals
cannot be ignored. You have to handle them, or the operating system will
abort the program.

But signals are rather uncommon. Most interfaces in C or POSIX return the
integer value of *-1* on errors and provide an additional way of retreiving
information about the actual error.

For floating-point operations, there are floating-point exceptions. These
can be queries with `fgetexceptflag()`. Divide by zero and you'll get
`FE_DIVBYZERO`, for example.

One of the oldest methods for communicating errors is by setting an error
code. Traditional error codes in C are provided in the header file
[*errno.h*][posix:errno_h]. There you'll find short identifier for all kinds
of possible errors: `ENOMEM` for out-of-memory situations, `EACCES` when
file access has been denied, or `EINVAL` if an incorrect argument was given
to a function. Each identifier maps to a positiv integer value, but the
actual value is system-dependent.

Some more-recent interfaces, such as POSIX threads, return error codes
directly, but most POSIX interfaces store them in the variable named `errno`.

#### The History of errno

The variable `errno` is quite old. I searched through the source code of
[The Unix Heritage Society][tuhs] and found its first in-code appearance in
[Unix Release V5][unixv5:errno]. According to the git history, the commit
was made on Nov 27 1974! Back then, there was no file named *errno.h.*
Instead `errno` was implemented as part of [`perror()`][posix:perror],
a function that outputs a descriptive string for each error code.

The error codes are even older than `errno`. The earliest error codes I
could find were in the source code of [Unix Release V4][unixv4:errno].
This file dates back to Aug 31 1973. The error codes were stored in the
field `u_error` of `struct user`. Even though the `errno` variable was
not present in the source code of V4, it's already mentioned on the
[man page for `perror()`][unixv4:man_perror]. This file dates back to
Nov 6 1973, so `errno` must have been introduced at some point between
Aug 31 and Nov 6 of 1973.

If you compare these old error codes to what is present on today's
Unix-like system, you'll find that they are mostly the same. In fact all of
the old error codes are still present. New releases and standards only added
more error codes when required.

So back in Unix Release V5, `errno` was an integer value. Error codes were
still returned in the field `u_error` of `struct user`, but automatically
copied to `errno`.

C was first standardized in the late 1980s and `errno` was part of this
standard. It was specified to be *a modifiable lvalue of type `int`,* just as
it was in Unix.

Compared to its prominent place in Unix, in the C Standard `errno` feels as
if it is either under-used or out of place. Only 3 error codes are part of
the C standard: `EDOM`, `EILSEQ` and `ERANGE`. Many traditional Unix
functions don't have error codes specified. For example, even though
`malloc()` is part of the C Standard Library, its error code `ENOMEM` is
not.

The implementation of `errno` remained of type `int` for roughly 22
years, and then... multithreading happend.

In a multithreaded program, having a global variable that signals the error
code is obviously a problem. In the time between one thread observing an
error (i.e., *-1* being returned from a function call) and reading the
error code from `errno`, an error on another thread could change the value
of `errno`. A global `errno` variable of type `int` is basically useless in
this environment.

The solution to this problem is illustrated in the example code shown below.

~~~ c
#ifdef	_THREAD_SAFE
extern	int *		__error();
#define	errno		(* __error())
#else
extern int errno;			/* global error number */
#endif
~~~

In a thread-safe program, `errno` is defined as macro that resolves to a
function call. The function returns a pointer to a thread-local copy of
the error variable and the `errno` macro internally dereference the pointer.
This way, accessing `errno` returns a thread-local error code.

The code snipped above has been
[taken from a rather confusing commit][freebsd:errno]
to FreeBSD. The commit was made on Jan 22 1996 and first shipped in FreeBSD
2.2.0.

Other systems, such as GNU or Linux, traditionally used different
implementations of POSIX and the C Standard Library, but have gone through
a similar transition.

#### Transactional errno

After this long excurse into history, let's get back to transactions. For
`errno` to be transaction-safe, all we have to do is to save its value
before it's being changed by any function. If a transaction aborts, we
reset `errno` to its original value, so the transactions always restarts
with the same error code. As the variable itself is already thread-local,
no concurrency control is required.

Building upon `malloc()` and `free()` support from the
[previous blog entry][post:20170623], let us first add an interface to safe
`errno` during the transaction.

~~~ c
void save_errno(void);
~~~

Too easy.

For the implementation, we add an errno value to the transaction context
as well as a flag showing the value's validity.

~~~ c
struct _tm_tx {
    jmp_buf env;

    unsigned long        log_length;
    struct _tm_log_entry log[256];

    bool errno_saved;
    int errno_value;
};
~~~

We further have to save `errno` and restore its value during a rollback.

~~~ c
void
save_errno()
{
    struct _tm_tx* tx = _tm_get_tx();

    if (tx->errno_saved) {
        return;
    }

    tx->errno_value = errno;
    tx->errno_saved = true;
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

    /* Restore errno */
    if (tx->errno_saved) {
        errno = tx->errno_value;
        tx->errno_saved = false;
    }

    /* Jump to the beginning of the transaction */
    longjmp(tx->env, 1);
}
~~~

We only save `errno` on the first time that `save_error()` gets called
in a transaction. That's the error code we're interested in. Later
invocations may not replace is. During a transaction's roll-back and
restart, we check if we saved `errno` and possibly restore it to its
original value.

So far, our transaction manager only supports one function that could
potentially modify `errno`, which is `malloc_tx()`. Modifying `malloc_tx()`
to save `errno` is trivial.

~~~ c
void*
malloc_tx(size_t size)
{
    save_errno();

    void* ptr = malloc(size);

    append_to_log(NULL, undo_malloc_tx, (uintptr_t)ptr);

    return ptr;
}
~~~

That's the same implementation as in the previous blog post, extended by
a call to `save_errno()`.

#### Summary

We've looked at `errno` and how to support it in a system transaction.

 - `errno` has a long history, dating back to the earliest Unix releases.
 - Today, `errno` is implemented as a thread-local variable.
 - Before executing an operation that modifies the value of `errno`,
   the original value has to be saved in the transaction context.
 - During a roll-back of the transaction, `errno` is restored to its
   original value.

As usually, you can find the full source code for this blog post
[on GitHub][source:20170630]. If you're interested in a more sophisticated C
transaction manager, take a look at [picotm][picotm].

If you like this series about writing a transaction manager in C, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

[man7:malloc_usable_size]:  http://man7.org/linux/man-pages/man3/malloc_usable_size.3.html
[picotm]:                   http://picotm.org/
[posix:errno_h]:            http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/errno.h.html
[posix:free]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/free.html
[posix:malloc]:             http://pubs.opengroup.org/onlinepubs/9699919799/functions/malloc.html
[posix:perror]:             http://pubs.opengroup.org/onlinepubs/9699919799/functions/perror.html
[post:20170505]:            {% post_url 2017-05-05-implementing-a-simple-transaction-manager-in-c %}
[post:20170509]:            {% post_url 2017-05-09-transactional-semantics-in-theory-and-practice %}
[post:20170512]:            {% post_url 2017-05-12-transaction-roll-back-and-serializability %}
[post:20170519]:            {% post_url 2017-05-19-writing-the-beginning-and-end-of-a-transaction %}
[post:20170526]:            {% post_url 2017-05-26-transactional-access-to-arbitrary-memory-locations %}
[post:20170601]:            {% post_url 2017-06-01-transactional-memcpy-and-resource-privatization %}
[post:20170609]:            {% post_url 2017-06-09-more-on-resource-privatization %}
[post:20170616]:            {% post_url 2017-06-16-transaction-logs %}
[post:20170623]:            {% post_url 2017-06-23-transactional-malloc-and-free %}
[source:20170630]:          http://github.com/tdz/blog_examples/tree/master/2017-06-30

[tuhs]: http://www.tuhs.org/


[unixv4:errno]:         http://github.com/dspinellis/unix-history-repo/blob/Research-V4-Snapshot-Development/sys/user.h#L39
[unixv4:man_perror]:    http://github.com/dspinellis/unix-history-repo/blob/Research-V4-Snapshot-Development/man/man3/perror.3#L28
[unixv5:errno]:         http://github.com/dspinellis/unix-history-repo/blob/Research-V5-Snapshot-Development/usr/source/s4/perror.c#L1

[freebsd:errno]:        https://github.com/dspinellis/unix-history-repo/commit/4a81bbc3e8796e1f18f1a5e8ba62836ad71025c2#diff-4c402341f93d93b29dc0f6eb539600f5R46
