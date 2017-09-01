---
layout:     post
title:      The Internals of Unix Pipes and FIFOs
date:       2017-09-01 14:30:00 +0200
tags:       [c, fifo, fileio, howto, io, ipc, linux, pipe, posix, tutorial, unix]
og-image:   tux.png
---

There is a series of blog posts examining the details of files and
file descriptors, etc on Linux and Unix. This installment continues with the
details of Unix FIFOs, how they work internally, and how they are used for
communicating between processes.

<!-- excerpt -->

#### First-In First-Out

A recent [series][post:20170728] [of][post:20170804] [posts][post:20170817]
on this blog describes the data structures surrounding Unix file I/O, such
as file buffers, open file descriptions, and file descriptors; how they work
together and interact with the file system. In these entries we mostly
focused on regular files, which have a representation in the file system
and a data buffer that is stored on a disk.

From the previous blog entries, remember how we opened a file twice
and got a number of unique and shared data structures.

``` c
int fd0 = open("/home/joe_user/my_file.txt", O_RDWR|O_CREAT, S_IRUSR|S_IWUSR|S_IRGRP);

int fd1 = open("/home/joe_user/my_file.txt", O_RDWR|O_CREAT, S_IRUSR|S_IWUSR|S_IRGRP);
```

![Data structures after opening `/home/joe_user/my_file.txt` twice.]({{ site.baseurl }}/img/unix2.png)

In this example we opened the file `/home/joe_user/my_file.txt` twice
and got two file descriptors, each refering to its own open file
description. Both open file descriptions refer to the same file buffer,
which holds the file's content. Using a FIFO is similar to this example,
but the involved buffer data structure is very different.

A *FIFO (first-in first-out)* is a data buffer with a write end and a read
end. Data is written on one end of the data buffer, and read from the other
end.

FIFOs were invented in the early 1970s by Douglas McIlroy. They were first
included in Unix Version 5, released in 1974. As we'll see below, FIFOs allow
for connecting the output of one program to the input of another program.
This makes them one of the building blocks of what is considered to be Unix.

#### Using and Creating FIFOs

Let's first create a FIFO and see what happens. FIFOs are also called
pipes, so the Unix function to create a pipe is called [`pipe()`][posix:pipe]
as well.

``` c
int pipefd[0];
pipe(pipe[fd]);
```

A call to `pipe()` creates two file descriptors, which are returned
in the array that `pipe()` takes as its argument. In the example, the file
descriptor for reading is stored in `pipefd[0]` and the file descriptor
for writing is stored in `pipefd[1]`.

After calling `pipe()`, the data structures look very similar to the
case where we opened a file twice. For the purpose of illustration,
process-local data structures are shown in light-blue color.

![Data structures after creating a pipe.]({{ site.baseurl }}/img/pipe1.png)

As mentioned we have created two new file descriptors. Each refers to an
open file description. Each open file description of the FIFO stores whether
it's on the read end or the write end. Both open file descriptions refer to
the same FIFO buffer, which the kernel just created.

Let's write to our new pipe. The interface for writing and reading is the
same as for files. Each end of a FIFO is represented by a file descriptor.
A call to [`write()`][posix:write] writes to the write end, a call to
[`read()`][posix:read] read from the read end.

``` c
const char string[] = "Hello world!";

write(pipefd[1], string, sizeof(string));
```

The FIFO buffer now contains the string *Hello world!*. We can read it
using the other file descriptor.

``` c
char buf[13];
read(pipefd[0], buf, sizeof(buf));

// buf now contains `Hello world!`
```

Reading transfers the content of the FIFO buffer into the read buffer and
removes the read data from the FIFO buffer.

Although the FIFO buffer might look similar to a file buffer at first, it
works in a different way.

First of all, there is no actual buffer on the disk drive. A FIFO buffer is
only contained in memory. So when the system shuts down, its content is gone.

A FIFO is a channel with a single direction and the order of data is
preserved. So when we write `Hello world!` on the write end, we will later
read `Hello world!` on the read end; but never `world! Hello` or
`Ehlol Rwold!`.

When accessing a file buffer, we can select the location of where the read
or write operation takes place. Because there is only a write end and a read
end for a FIFO, it's not possible to select the location. If we wrote
*Hello world!* in the example, there's no way to only read *world!* without
reading *Hello* first.

#### Duplicating and Closing FIFOs

A file can be duplicated by simply copying it to a new location in the file
system. For FIFO buffers it's not possible to duplicate them in any similar
way. All we can do is to duplicate the file descriptor using
[`dup()`][posix:dup].

``` c
read_fd = dup(pipefd[0]);
write_fd = dup(pipefd[0]);
```

These duplicated file descriptors work exactly the way that the original
file descriptors worked.

As with all file descriptors, the read and write ends of a pipe are closed
using [`close()`][posix:close]. The kernel releases the FIFO buffer after
all its file descriptors have been closed.

``` c
close(read_fd);
close(write_fd);
```

#### Inter-Process Communication using FIFOs

Now that we've looked at the interface, let's see what FIFOs are actually
used for. By far the most common usage of FIFOs is to communicate between
processes.

In the previous example, we already created a FIFO with a call to `pipe()`.
Let's now duplicate the process using [`fork()`][posix:fork].

``` c
int pipefd[0];
pipe(pipefd);

fork();
```

As we have seen in previous blog entries, a call to `fork()` only copies the
process-local file descriptors, but the underlying buffers an open file
descriptions are shared by the old and the new process. So our data
structures now look like this.

![Data structures after creating a pipe.]({{ site.baseurl }}/img/pipe2.png)

It might seem confusing at first, but what this diagram shows is a FIFO
with read and write end, and two processes that each have a file descriptor
refering to either end of the FIFO. The old process' data structures are
again shown in light-blue color, the new process' data structures are shown
in orange color.

We can now communicate between processes using `read()` and `write()` as in
the examples above. If we write to the FIFO in the old process

``` c
const char string[] = "Hello world!";

write(pipefd[1], string, sizeof(string));
```

the new process can read the data from the FIFO.

``` c
char buf[13];
read(pipefd[0], buf, sizeof(buf));

// buf now contains `Hello world!`
```

Currently both processes can access both ends of the FIFO buffer. In
practice each process would close one end of the FIFO, such that there
is one writer and one reader. For bi-directional communication, there would
be a second FIFO and processes would act in opposite roles.

#### An Excursus on the Shell

Using FIFOs, a Unix terminal allows us connect the input and output of
different programs.

``` sh
cat /home/joe_user/my_file.txt | less
```

This command starts two programs `cat` and `less`. The call to `cat` reads
the content of the file `/home/joe_user/my_file.txt` and writes it to its
standard output. The call to `less` reads text from its standard input and
display it to the user.

There's also a pipe symbol `|` between them. This instructs the Unix shell
to connect both programs using a FIFO. The shell creates a pipe, creates the
two processes and makes the FIFO's write end the standard output of `cat`.
The FIFO's read end becomes the standard input of `less`. After that, each
process executes either the `cat` or the `less` program. Anything written
by `cat` goes into the FIFO, and anything read by `less` comes out of the
FIFO.

#### Named Pipes

With the pipes that we've seen so far, there has to be some kind of
relationship between communicating processes. One must be a copy of the
other, or both must have the same parent process. Otherwise it would not
be possible for them to refer the same FIFO buffer. So if we have two
unrelated processes, they cannot communicate this way.

The solution to this problem are named pipes. A *named pipe* is a special
kind of file that has a name in the file system, but refers to a FIFO
buffer.

We can create a named pipe with a call to [`mkfifo()`][posix:mkfifo].

``` c
mkfifo("/home/joe_user/my_fifo", 0);
```

Here we create a named pipe with the name `/home/joe_user/my_fifo` and the
default file-mode flags. It's path is now present in the file system.

And just like with a regular file, a process opens a named pipe with a call
to `open()`.

``` c
int fd0 = open("/home/joe_user/my_fifo", O_RDONLY);

int fd1 = open("/home/joe_user/my_fifo", O_WRONLY);
```

After opening the named pipe at least once for reading and at least once for
writing, it is usable. Because these calls to `open()` can happen in any
process with permission to access the file path, arbitrary programs can
communicate using a named pipe.

The big difference to regular files is that the name pipe refers to a FIFO
buffer. There is still no file buffer on the disk and as soon as the system
shuts down, any un-read content of the named pipe is lost.

#### Summary

In this blog post, we discussed usage and data structures of FIFOs on
Unix systems.

 - FIFOs, also known as pipes, are data buffers with a write end and a
   read end.
 - Data written to a FIFOs write end is read from the read end in the
   same order.
 - A pipe is created using the `pipe()` function.
 - Reading and writing on a FIFO is performed with `read()` and `write()`
   in the same way as with regular files.
 - It is not possible to skip or ignore data while reading.
 - Pipes are used for communication between related programs.
 - A named pipe is a FIFO with a representation in the file system.
 - Names pipes can be opened by arbitrary processes and allow unrelated
   programs to communicate with each other.

If you like this blog post about the basics of Unix file I/O, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

[posix:close]:              http://pubs.opengroup.org/onlinepubs/9699919799/functions/close.html
[posix:dup]:                http://pubs.opengroup.org/onlinepubs/9699919799/functions/dup.html
[posix:fork]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html
[posix:mkfifo]:             http://pubs.opengroup.org/onlinepubs/9699919799/functions/mkfifo.html
[posix:open]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/open.html
[posix:pipe]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/pipe.html
[posix:read]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/read.html
[posix:write]:              http://pubs.opengroup.org/onlinepubs/9699919799/functions/write.html
[post:20170728]:            {% post_url 2017-07-28-data-structures-of-unix-file-io %}
[post:20170804]:            {% post_url 2017-08-04-unix-hard-links-soft-links-and-files %}
[post:20170817]:            {% post_url 2017-08-17-file-descriptors-during-fork-and-exec %}
