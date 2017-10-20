---
layout:     post
title:      Scaling Lock Performance
date:       2017-10-20 19:00:00 +0200
tags:       [benchmark, locking, picotm, stm]
og-image:   benchmark.png
---

Release 0.7 of [picotm][picotm] will feature several major improvements of
the locking code. In this blog post, we're going to look at each change and
how it impacts performance and scalability. We'll go from exclusive locks to
reader/writer locks to exclusive mode to a dedicated lock manager.

<!-- excerpt -->

#### The Test Setup

[Picotm][picotm] is a transaction manager for system-level software. For
example, we can write transactions that concurrently perform operations on
file descriptors and file buffers, and picotm will detect reader/writer
conflicts and errors for us. In the case of a conflict, one of the
conflicting transactions has to roll back into its original state and retry.
In the case of an error, picotm executes application-specific error handling
code.

Our tools for measuring the performance is
[`picotm-perf`][github:picotm-perf]. It's a
[simple test tool and comes with a benchmark script][post:20170927]
for running a large set of tests.

Tests within `picotm-perf` operate on transactional memory. Each test

 1. begins a transaction,
 2. performs a number of load operations from shared memory,
 3. performs a number of store operations into shared memory, and
 4. commits the transactions.

All load and store locations are randomized. The shared memory is a memory
buffer that is used for all transactions' loads and stores. With only 1024
bytes it's fairly small, so conflicts are more or less guaranteed.

Picotm-perf is a synthetic benchmark; all results are only meaningful
relative to each other, but they probably don't represent real-world
performance in any way. Change any parameter and the results could be
entirely different.

The test environment is outlined in the PDF files with the results. Note
that the test machine has 4 logical processor cores, but only 2 physical
cores. The performance won't scale linearly with the number of cores on
such a system.

#### The Original Code of picotm 0.6

Let's start with the original implementation that shipped with picotm's
previous release 0.6. It divides the shared memory buffer into small records
of 8 bytes and locks each record with an exclusive lock. This means that two
transactions observe a conflict when both access the same record concurrently;
whether it's a load or a store operation does not matter.

The [benchmark results][site:results_v0_6_pdf] (pg. 3-8) for random memory
access reflect this. Independently from the number of loads and stores, all
graphs look the same. Here's an example of a transaction with 50 loads and
50 stores.

![picotm v0.6, I/O pattern random, 50 loads, 50 stores]({{ site.baseurl }}/img/results-v0.6.png)

The blue bars and the scale on the left show the number of committed
transactions per second; the red bars and the right scale show the number
of restarts per second.

We already mentioned the effect of exclusive locks on concurrency. For a
single transaction we start with around 50,000 commits per seconds, but go
down as we add concurrent transactions ot the test. And that's even the case
if we only run transactions with load operations, which don't require
exclusive locks.

Another problem that becomes obvious from the test results is the extremely
high number of restarts. There are up to 4 million restarts per second!

These results gives us our first targets for improvement: enable concurrent
access to memory records for transactions that only perform load operations,
and reduce the number of restarts.

#### Adding Reader/Writer Locks

[Pull request #140][github:pr140] on GitHub converts the transactional-memory
module to use reader/writer locks. These locks are already provided by picotm
and used throughout the file-I/O code to protect concurrent access to files
and file buffers.

Here's an example from the [test results][site:results_limit_nretries_pdf]
for reader/writer locks. The transaction again consists of 50 load and 50
store operations.

![With R/W locks, I/O pattern random, 50 loads, 50 stores]({{ site.baseurl }}/img/results-tm-rwlocks.png)

The results are two-fold. We have reduced the number of restarts by a factor
of 10 to 15 to something in the 200 thousands. Unfortunately, this didn't
increase the number of commits per seconds. Quite the opposite actually. With
more than one transaction present, the result drops to a few 1.000 commits
per second.

An exception is the case of the load-only transactions, shown on page 8 of
the [PDF file][site:results_limit_nretries_pdf]. With the new reader/writer
locks, reader transactions finally start scaling.

At this point it's worth remembering that our shared memory buffer is only
1 KiB in size. With a buffer size that small, as soon as we have write
operations in our transaction, conflicting access among transactions are
likely.

The low number of concurrent commits is the result of the small buffer
size and the interactions between concurrent transactions, competing for
locks. We have hit a pathological corner case. If we could *guarantee*
progress to some transactions, things might improve. One simple way of
doing this is a limit on the number of restarts per transaction.

#### Adding Exclusive Mode

Exclusive mode is a variant of irrevocability. In exclusive mode, we
run one single transaction exclusively. All other transactions are
blocked and consequently conflicts among transcations cannot occur.

[Pull request #141][github:pr141] on GitHub enables exclusive mode on top
of the reader/writer patch set. After a transaction has reached 10 retries,
picotm stops all other transactions, which now have to wait until completion
of the exclusive one. The limit has been choosen arbitrarily and its optimum
depends on the workload and access patterns.

The complete results are in [this PDF file][site:results_tm_rwlocks_pdf].
Here's an example.

![With exclusive mode, I/O pattern random, 50 loads, 50 stores]({{ site.baseurl }}/img/results-limit-nretries.png)

The results look really promising. By limiting the number of restarts per
transaction, we have reduced the overall number of restarts by factor 10
to 100,000 to 140,000 restarts per second. At the same time we increased
the number of concurrent commits by a factor of 10 to up to 20,000.

On top of the scalability improvements, exclusive mode offers another
important feature: transactions are now guaranteed to commit and our
overall system will make progress in any case. Imagine a scenario where
transactions constantly conflict with each other, restart, and conflict
again without ever getting out of this pattern. This is called a *livelock.*
With the previous implementation this was very much possible; with the
new implementation not so. With this patch set, we did not just improve
scalability, we resolved a whole class of concurrency problems.

Even with our current patch set we still observe 100,000 to 140,000 restarts
per second. These restarts are required to avoid cyclic dependencies among
transactions, as all locks held by a transaction are released if the
transaction restarts. By simply never waiting, no cyclic dependencies, and
therefore no deadlocks, can occur.

Our next improvement will try to reduce the number of restarts by not
restarting after all. That's what a lock manager does.

#### Adding a Lock Manager

The lock manager is a software component within picotm that

 - maintains lists of transactions waiting for each reader/writer lock, and
 - schedules waiting transactions when a lock becomes available.

The [actual implemenation][github:pr142] is quite complex and worth a
separate blog post. The overall algorithm can be summarized like this

 1. A transaction tries to acquire a reader/writer lock.
 2. If the transaction fails to acquire the lock, it instructs the lock
    manager to make it wait for the lock to become available.
 3. The lock manager inserts the transaction into a waiting queue for
    the lock.
 4. When the lock becomes available, the unlocking transaction instructs
    the lock manager to select one or more of the waiting transactions
    and wake them up.
 5. In the waiters
    1. the woken-up transactions try again to acquire the lock.
    2. If a transaction has not been woken up after a certain timeout, it
       wakes up automatically and a tries to acquire the lock.
 6. Only if a transaction fails a second time to acquire the lock, it
    restarts; otherwise it continues as if lock acquisition had been
    successful on the first try.

Point 5.2 is required to avoid cyclic dependencies, and therefore deadlocks,
among waiting transactions. If the waiting time has an upper bound, cyclic
dependencies will be resolved as soon as the timeout has been reached and
one of the transactions restarts.

Lock management gives us a new quality of locking: we now know which
transaction waits for which lock, we can keep statistics about each
transaction's locking patterns, and we have a central place that coordinates
locks among transactions.

At the moment [the results][site:results_lock_manager_pdf] are again
two-fold. Here's our example result.

![With lock management, I/O pattern random, 50 loads, 50 stores]({{ site.baseurl }}/img/results-lock-manager.png)

Compared to the previous patch set, we have reduced the number of restarts
by another factor of 8 to 14.000 to 16.000 restarts per second. Compared to
the 4 million restarts per second with picotm 0.6 that's an overall
improvement by factor 250. Amazing!

Unfortunately, lock management actually decreased the number of
commits per second significantly. The reason is in the high overhead
of maintaining the waiter lists and suspending the waiting transactions,
which involves a number of system calls. That's significant compared to
the fast load and store operations that are required for simply restarting.

#### Where To Go From Here?

picotm 0.7 will feature all of the improvements we discussed in this blog
post. Reader/writer locks and the exclusive mode are definitely useful
changes.

The lock manager will be available, but probably not be used for the
most part. For these memory-only transactions of our test program, maintaining
waiter lists and suspending transactions makes no sense, because simply
restarting conflicting transactions in exclusive mode gives much better
results.

The situation changes with longer transactions that may even have a higher
system-wide overhead. For example, file I/O requires system calls for
reading and writing data in the file buffer. The longer such a transaction
runs, the more likely it is that it reaches a tiping point where waiting
is the better option over restarting immediately.

Our current lock manager's transaction scheduler always wakes up the longest
waiting transaction when the lock becomes available. That's probably not
optimal in most situation. By maintaining better locking statistics for
each transaction, we can make better decisions about which transaction to
wake up. There are zillions of strategies and tuning parameters to explore!

#### Summary

In this blog post, we've looked at the impact of different locking patch sets
to the scalability and performance of picotm's transactions.

 - Exclusive locks don't scale well, lead to a large number of conflict
   restarts, and don't allow for concurrent readers.
 - Reader/writer locks improve concurrency in setups with many reader
   transactions and only few conflicts. They are not that helpful if
   conflicts with writer transactions are likely.
 - Switching transactions to an exclusive mode after they reached a limited
   number of restarts further improves scalability and prevents livelocks.
 - Lock management can significantly reduce the number of restarts, but
   imposes overhead on the transactions.

If you like this blog post, please subscribe to the RSS feed, follow on
Twitter or share on social networks.

[github:picotm-perf]:               http://github.com/picotm/picotm-perf
[github:pr140]:                     http://github.com/picotm/picotm/pull/140
[github:pr141]:                     http://github.com/picotm/picotm/pull/141
[github:pr142]:                     http://github.com/picotm/picotm/pull/142
[picotm]:                           http://picotm.org/
[post:20170927]:                    {% post_url 2017-09-27-benchmark-visualization-with-latex-and-gnuplot %}
[site:results_v0_6_pdf]:            {{ site.baseurl }}/assets/pdf/results-v0.6.pdf
[site:results_tm_rwlocks_pdf]:      {{ site.baseurl }}/assets/pdf/results-tm-rwlocks.pdf
[site:results_limit_nretries_pdf]:  {{ site.baseurl }}/assets/pdf/results-limit-nretries.pdf
[site:results_lock_manager_pdf]:    {{ site.baseurl }}/assets/pdf/results-lock-manager.pdf
