---
layout:     post
title:      A picotm Demo Application
date:       2018-01-12 10:00:00 +0100
tags:       [picotm]
og-image:   share2.png
---

Happy new near to you! I spent the first week of 2018 on writing a [demo
application][picotm:demo] for [picotm][picotm]. The intention is to show
what picotm has to offer and how system-level transactions work in a real
program. In this blog post we'll go through it and see what it does.

<!-- excerpt -->

The demo application reads random data from Linux, puts it into several
memory buffers, and visualizes the buffers' content on screen. This is
split among multiple threads, which segment the application into three major
parts: an input thread, a number of processing threads, and a (text-mode)
visualization thread.

As much as possible, this functionality is implemented using picotm
transactions. All the shown transaction functionality is part of picotm and
ready to use out-of-the-box.

But let's start with a screen shot. At first, all buffers are empty.

![All application buffers are empty at first.]({{ site.baseurl }}/img/demo1.png)

#### The Input Thread

First of all there's an input thread that periodically reads data from
the operating system. It does so by opening `/dev/urandom` and calling
`read()` on it once per second.

Here's the opening code. It imports the filename into the transaction,
opens the file and exports a file descriptor from the transaction to
main memory.

~~~ c
static int
open_input_file(const char* filename)
{
    int fd;

    picotm_begin

        // Privatize the memory of the argument and filename
        privatize_c_tx(filename, '\0', PICOTM_TM_PRIVATIZE_LOAD);

        // Open input file
        int tx_fd = open_tx(filename, O_RDONLY);

        // Export the file descriptor from the transaction
        store_int_tx(&fd, tx_fd);

    picotm_commit
        int res = recover_from_tx_error(__FILE__, __LINE__);
        if (res < 0) {
            return -1;
        }
        picotm_restart();
    picotm_end

    return fd;
}
~~~

Remember that we don't require error detection in transactional code. This
is done fully automatically by the transaction framework. If anything goes
wrong, the framework rolls back the transaction and jumps to the recovery
code that sits between the commit and end statements. From there, we can try
to repair the error and restart the transaction.

Next is the reading code. Again it's implemented as a transaction.

~~~ c
struct queue_entry {
    struct txqueue_entry entry;
    struct hdr msg;
};

static struct queue_entry*
read_file(int fd)
{
    struct queue_entry* entry;

    picotm_begin

        struct queue_entry* tx_entry = malloc_tx(sizeof(*entry));

        // Read message header from input stream. The value of `fd` is a
        // constant on the stack; no need to load or privatize. The first
        // 4 byte are considered meta data.
        read_tx(fd, &tx_entry->msg, 4);

        // Read data into buffer. With `read_tx()` the buffer `msg.buf` is
        // automatically privatized by the TM module.
        read_tx(fd, tx_entry->msg.buf, tx_entry->msg.len);

        // Export message from transaction context.
        store_ptr_tx(&entry, tx_entry);

    picotm_commit
        int res = recover_from_tx_error(__FILE__, __LINE__);
        if (res < 0) {
            return NULL;
        }
        picotm_restart();
    picotm_end

    return entry;
}
~~~

Not trying to bore you with details, this transaction allocates a message
structure named `queue`, reads the random data into the structure, and exports
it from the transactional context. Again, if anything goes wrong, the
transaction framework detects the error for us and goes to recovery. Because
transactions cleanly separate the error detection from the recovery, we
can re-use the *exact* recovery code from the opening transaction.

We now have a packet containing random data. Each data packet is forwarded
to one of multiple processing threads. The forwarding transaction is
trivial (as it should be).

~~~ c
picotm_begin
    struct txqueue* queue = txqueue_of_state_tx(&q->queue);
    txqueue_push_tx(queue, &entry->entry);
picotm_commit
    int res = recover_from_tx_error(__FILE__, __LINE__);
    if (res < 0) {
        goto out;
    }
    picotm_restart();
picotm_end
~~~

The variable named `entry` is the data packet we just created, the variable
named `queue` is a transactional queue that connects the input thread with a
processing thread.

#### The Processing Threads

Processing threads receive data from the input thread for further
processing. Each maintains an in-memory buffer, where it stores the
received data.

The following code removes the input thread's data packet from the
transactional queue. This is the ownership hand-over step of the data and the
memory accociated with it. Once the processing thread reads the packet,
it owns it.

~~~ c
bool continue_loop;

do {

    continue_loop = false;

    picotm_begin

        // Acquire transactional queue for queue state and get
        // next message from queue.
        //

        struct txqueue* queue = txqueue_of_state_tx(&q->queue);
        if (txqueue_empty_tx(queue)) {
            goto commit;
        }
        struct queue_entry* entry =
            queue_entry_of_txqueue_entry_tx(txqueue_front_tx(queue));

        // Copy message buffer into correct field and fill trailing
        // bytes with 0.
        //

        uint8_t* field = buf->field[entry->msg.off];
        memcpy_tx(field, entry->msg.buf, entry->msg.len);
        memset_tx(field + entry->msg.len, 0, 256 - entry->msg.len);

        // Remove message from queue and free memory.
        //

        txqueue_pop_tx(queue);
        free_tx(entry);

        // Continue loop until queue runs empty
        store__Bool_tx(&continue_loop, true);

    commit:
    picotm_commit
        int res = recover_from_tx_error(__FILE__, __LINE__);
        if (res < 0) {
            return;
        }
        picotm_restart();
    picotm_end

} while (continue_loop);
~~~

The data is then copied into the memory buffer `buf->field`. The processing
thread doesn't really do much processing, but could in a real-world
application. The transaction applies Transactional Memory for the copy
operation (i.e., `memcpy_tx()` and `memset_tx()`). Our output thread
will therefore be able to read the buffer content without interfering
with the processing.

In the final step, the transaction removes the data packet from the
transactional queue and frees its memory.

Again, you can see how we reuse the error recovery. Recovery strategies
hardly change within an application, so the recovery code should be
separate from the error detection.

At this point, we have received data from an input source forwarded it
to another thread, processed it, and stored the results in main memory. A
fairly common pattern in most applications.

#### The Output Thread

The output thread, actually the application's main thread, visualizes the
buffer content. It periodically reads from each buffer and reduces the
buffer's content to a string of hexadecimal numbers. These strings are then
displayed on the screen.

Just as in the case of the processing thread, access to each buffer is
performed using Transactional Memory. Conflicts among output and processing
threads are resolved automatically. Over time each buffer fills with content;
a process that the user can follow by watching the screen. After a while the
application buffers contain data.

![After a while the application buffers contain data.]({{ site.baseurl }}/img/demo2.png)

#### Summary

In this blog post, we've looked picotm's demo application.

 - The demo application reads random data into memory buffers and
   visualizes the buffers' content on the screen.
 - Input, processing and visualization is performed on different
   threads.
 - Transactional code is employed for communication among threads
   and detecting errors.

The full source code for the demo is available in a
[git repository][picotm:demo] on GitHub.

If you like this blog post, please subscribe to the RSS feed, follow on
Twitter or share on social networks.

[picotm]:       http://picotm.org/
[picotm:demo]:  http://github.com/picotm/picotm-demo
