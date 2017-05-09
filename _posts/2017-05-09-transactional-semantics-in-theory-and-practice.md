---
layout: post
title:  Transactional Semantics in Theory and Practice
date:   2017-05-09 18:00:00 +0200
tags:   [atomicity, concurrency control, consistency, durability, error handling, isolation, practice, semantics, theory]
---

In the [previous installment][simpletm1], we implemented a simple transaction
manager, but we didn't really say what it means to run code as a transaction,
which features are considered *transactional* and which aren't.

In this blog post, we're first going to examine the theoretical priciples of
transactional semantics, and afterwards look at how to put them into practice.

<!-- excerpt -->

#### Why Bother?

It's certainly fair to ask why we should bother.

Image a software program of your choice. If we take a step backwards from the
actual code, we can describe the software as a [state machine][statemachine].

As the name implies, the state machine is always in a *state.* What the state
consists of depends on the software. For example, a chess game might be in a
state where it waits for the player's next move. While it does this, it
doesn't change. At the software level, think of state as a combination of the
values of all data structures.

Now something happens. Maybe the user pressed a key, or a hardware sensor
fired, or a new packet arrived on the network. Our state machine responds
according to which state it is in. When it does so, it performs a specific
action and moves into a new state. This movement from one state to the next
while performing an action in between is called a *transition.*

In a real-world program, the action is implemented by software. When the state
is in data structures, the action is in algorithms.

When the software is in a specific state, a specific input triggers a specific
action, which transitions the software into a new state. Normally all states
should be well-defined and transitions should always move the state machine
from one well-defined state to another well-defined state.

So far, so good. But back in the real world, not all state can handle all
input; and if it can, the associated action can be faulty. When this happens,
the state machine goes from a well-defined state into an undefined state.
What happens next is not specified.

We call this a software bug.

If we could implement our transition such that it detects incorrect input or
internal problems automatically, and has the option of going back to the old,
well-defined state; we'd be able to rule-out a large number of incorrect
transitions into undefined states.

This is were the transaction concept comes in handy.

#### The ACID Properties

Transactional semantics are defined by four properties, which the transaction
manager guarantees to its transactions. Those are called the ACID properties,
from their initial letters. If you had at least a minimal exposure to database
theory, you have already seen them before.

The ACID properties are

 - Atomicity,
 - Consistency,
 - Isolation, and
 - Durability.

Let's go through them one by one. Consistency directly follows from our
previous discussion of states and transations, so let's start with this.

*Consistency* describes a requirement to move from one well-defined state
to another. For example, the HTML documents of a website refer to each other
via hyper links. If one of the document's files gets renamed, the consistency
constraints could require that all links in all documents refering to the
renamed file should be updated to the new filename.

What is considered well-defined depends on the software at hand; therefore
guarantee-ing consistency always requires some help from the application. The
transaction system can only help with implementing this. In our example, if
the rename-and-update transaction simple ignores some of the HTML documents
for an update, the result is probably inconsistent.

To guarantee consistency, it's obvious that we have to guarantee
atomicity as well. *Atomicity* specifies that we either do all of
the operations contained in a transaction, or none. That's something, the
transaction system can help a lot with. In the example with the HTML
documents, the transaction system would ensure that all documents are updated,
or, if updating is not possible, revert to the old state before the rename.

So far we've only considered a single transaction. Now let's add another,
concurent transaction into the mix. This brings us to isolation. The
*isolation* requirement specifies that the effects of a transaction shall
not interfere with the effects of another, concurrent transaction.

To give an example of *isolation*, if both of our transactions rename
filenames of HTML documents and update the references, they should not
overwrite each others changes. In most cases the cumulative effect of
both transactions should look as if the transactions had been executed
one-after-the-other.[^1]

Finally, we have durability. *Durability,* also known as persistence,
specifies that effects of a transaction persist, once the transaction
committed the effects. A later transaction will always see these changes and
not a previous state missing them. For our HTML example this means that once
a document has been successfully renamed, a later transaction will not see
the old filename.

Durability is often not achievable by the transactional system alone. It
requires an environment that ensures certain constaints.
For example, if an HTML document gets renamed and suddenly the disk drive
fails, the rename can get lost. The environment must apply sophisticated
back-up and replication strategies to avoid this.


#### Putting Theory into Practice

The ACID properties are most of all requirements, but not practical
solutions. This is the job of the transaction manager.

The transaction manager provides the transactions with a number of
operations. For each operation it implements two features. These
are

 - error handling, and
 - concurrency control.

Who really cares about handling all errors correctly? Handling means
*detecting* every error and always being able to move back into a previous
consistent state. The latter is called *recovery.*

Common sense says that error handling is the worst part of each software.
Errors are hard to test for and rarely happen. The error handling is
therefore the code that gets executed the least. Recovery is often complicated
as it requires significant effort to being able to back in state. In
traditional software, this is almost impossible.

Transaction managers implement all logic required to detect every possible
error for each of its operations, and contain the logic for going back into
a previous state; therefore recovering from the detected error.

This provides a significant benefit over traditional approaches where
application logic and error handling are written side by side. Because the
transaction manager implements error handling once, all transactions
automatically employ it. Therefore all transactional application logic
automatically employs it as well. This gives us the atomicity and consistency
requirements from the ACID properties.

With a good transaction manager, all the error handling that's left for the
application is the pathological cases, when the software reaches hardware
limits or sees hardware failure. Even in these cases, error detection is still
provided by the transaction manager and the application at least gets a
chance of reporting the error to an administrator and shutting down safely.

*Concurrency control* describes the logic that controls resource access and
protects resources against concurrent access from multiple transactions. A
resource can be any data structure that transactions might want to use, maybe
a file or shared memory.

There's a lot to concurrency control, but you can think of it as locking.
Locking happens to be the most common implementation.

Controlling resource access gives us the atomicity, consistency and isolation
requirements of the ACID properties. Without concurrency control, transactions
would constantly overwrite each others effects and operate on inconsistent,
half-committed state changes made by other transactions.

#### Summary

There was quite of theory on this blog post. We first

 - looked at software as a set of states that are connected by transitions.

We can make these transitions far less error prone by guarantee-ing the
properties of

 - atomicity,
 - consistensy,
 - isolation, and
 - durability.

This is called a transaction. A transaction manager helps with implementing
these guarantees by providing

 - error handling, and
 - concurrency control.

Next time, we'll get back to [simpletm][simpletm1], our toy transaction
manager. If you're interested in a complete transaction manager, take a
look at [picotm][picotm].

#### Footnotes

[^1]:   This *one-after-the-other* requirement is called serializability.
        We'll examine this in a later post.

[picotm]:       https://picotm.github.io/
[simpletm1]:    {% post_url 2017-05-05-implementing-a-simple-transaction-manager-in-c %}
[statemachine]: https://en.wikipedia.org/wiki/Finite-state_machine
