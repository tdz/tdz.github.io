---
layout:     post
title:      File Descriptors During fork() and exec()
date:       2017-08-17 16:00:00 +0200
tags:       [c, bsd, exec, file, fork, howto, io, linux, posix, tutorial, unix]
og-image:   tux.png
---

In the previous blog posts, we examined the relationship between files,
file names, file descriptors, and open file descriptions. This time we look
at what happens to file descriptors when we start a new program by calling
`fork()` and `exec()`.

<!-- excerpt -->

#### Opening a File

As in previous examples, let's [open][posix:open] a file on the file system.

``` c
int fd0 = open("/home/joe_user/my_file.txt", O_RDWR|O_CREAT, S_IRUSR|S_IWUSR|S_IRGRP);
```

This creates a file descriptor, an open file description, and a file buffer.
The following graph illustrates this, highlighting the per-process structures
in light blue color. A [previous blog post][post:20170728] describes the
open procedure in detail.

![Data structures after opening `/home/joe_user/my_file.txt` once.]({{ site.baseurl }}/img/unix6.png)

#### File Descriptors and Process Creation

Right now there's only one Unix process in our example. New processes are
created by a call to [`fork()`][posix:fork]. Invoking `fork()` duplicates
the calling process, such that the new process is a copy of the caller.

What happens to the file descriptors during a fork? They are part of the
process, so they are copied for the new process. So is the file-descriptor
table.

Let's call `fork()` in our example process.

``` c
fork();
```

We now have two processes and each has its own file descriptors and
file-descriptor table.

![Data structures after calling `fork()`.]({{ site.baseurl }}/img/unix7.png)

The process-specific data structures of the original process are still
displayed in a light blue color. The new process' data structures are
displayed in orange color. The file descriptor has been duplicated and
refers to the new process' file-descriptor table. The file descriptor
table itself is part of the pre-process state and has also been copied
by the call to `fork()`.

An important detail of the design of Unix is that the open file
description and the file buffer are are kernel data structures. Therefore
they are not duplicate by `fork()`. Consequently, whenever one of the
processes reads from or writes to it's local file descriptor, the effects
in the open file description and file buffer are seen by the other
process.

On the other hand, operations that work solely on file descriptors, such
as [`close()`][posix:close] or [`dup()`][posix:dup] don't effect the other
process. So if either process closes its file descriptor, the other
process doesn't notice.

#### Executing a Different Program

After a call to `fork()`, both the original and the new process continue
execution at the next instruction after `fork()`. To really execute a
different program, a process invokes [`execl()`][posix:execl].

The `execl()` system call instructs the kernel to replace the user-space
program of a process with the program stored in an executable file. A
call to `execl()` takes the path of the executable file and a list of
command-line arguments for the new program.

Let's call `execl()` in the new process that we just created with `fork()`.

``` c
execl("/bin/my_executable", NULL);
```

Our new process will now abandon the original program and execute the
program stored in `/bin/my_executable`.

What does this mean for the process' file descriptors and
file-descriptor table? Nothing! They are still in place and working as
before. Remember that the user-space file descriptor is really just an
integer index into the file-descriptor table. If the new program reads and
writes using the existing file-descriptor value, it reads and writes the
file opened by the old program. So even after executing a new program,
our diagram looks as before.

![Data structures after calling `fork()`.]({{ site.baseurl }}/img/unix7.png)

The good thing about this is that it's a nice, generic way of
communicating between programs. For example, when starting a new program
from the Unix terminal, the terminal sets up the program's default input
and output file descriptors to refer to what the terminal itself uses;
let's say keyboard and text screen. If instead the terminal uses a serial
port for input and output, a new program started from the terminal also
uses the serial port. This way the new program inherits these settings
from the terminal. And likewise, as a user we can direct a program's default
input and output file descriptors to anything that looks like a file, such
as pipes, sockets or regular files.

Unfortuantely, there is also one major drawback of this whole design.
It's too easy to leak file descriptors into a newly executed program. That's
what happened [here][medium:ofinjch].

The only standardized file descriptors are those for default input and
output, and the one for error reporting. They are the file descriptors
*0,* *1* and *2* respectively. The newly executed program does not know
about any other file descriptors opened by the process' old program.

This is bad for two reasons. First of all it unnecessarily consumes
resources. The maximum number of file descriptors per process is limited;
typically to *1024.* A program can run out of available file descriptors
quickly if it doesn't close file descriptors after their final use.

An even more sever problem is that the old program can leak information
into the new program that the new program is not supposed to see. A file
descriptor might refer to a file with sensitive data that only the original
program was supposed to access. After the call to `execl()` the new program
would be able to read this data as well.

To prevent this, file descriptors have to be closed *before* the new program
gets loaded by the kernel. Fortunately, there's a simple way of ensuring
this constraint: the `O_CLOEXEC` flag.

At the very beginning of this blog entry, we opened a file with the following
code fragment.

``` c
int fd0 = open("/home/joe_user/my_file.txt", O_RDWR|O_CREAT, S_IRUSR|S_IWUSR|S_IRGRP);
```

By or-ing `O_CLOEXEC` into the flags argument, the returned file descriptor
will be closed before the process, or any forked process, starts executing a
new program.

``` c
int fd0 = open("/home/joe_user/my_file.txt", O_RDWR|O_CREAT|O_CLOEXEC, S_IRUSR|S_IWUSR|S_IRGRP);
```

How would our example look if we opened the file like this? During the fork
the file descriptor and file-descriptor table are copied into the new
process. The file descriptor carries the `O_CLOEXEC` flag in its entry in the
file-descriptor table, but nothing else changed.

The big change comes when the new process executes `execl()`. Before loading
the new program into the process, the kernel goes through the process'
file-descriptor table and closes all file descriptors that have the
`O_CLOEXEC` flag set. Afterwards the resulting file data structures look
like this.

![Data structures after calling `execl()` with `O_CLOEXEC` set.]({{ site.baseurl }}/img/unix8.png)

The `O_CLOEXEC` flag can also be set and cleared with a call to
[`fstat()`][posix:fstat] after the file descriptor has been created. But
setting it later creates the possibility of meanwhile calling `execl()` and
leaking file descriptors.

In practice, the safest strategy is to *always* set `O_CLOEXEC` when a new
file descriptor gets created; and explictly clear the flag right before the
call to `execl()` for those file descriptors that are supposed to be handed
over to the new program.

#### Summary

In this blog post, we examined the behavior of Unix file-I/O data structures
during `fork()` and `execl()`.

 - A call to `fork()` duplicates the calling porcess.
 - During the fork operation, the kernel copies the process' file descriptors
   and file-descriptor table, but not the underlying open file descriptions
   and file buffers.
 - Processes can communicate with each other by using the shared open file
   descriptions and file buffers.
 - A call to `execl()` starts a new program within the calling process.
 - By default, executing a new program has no effect on file-I/O data
   structures. The new program will have access to all file descriptors,
   open file descriptions and file buffers left open by the old program.
 - Leaving file descriptors open before executing a new program can leak
   resources and/or sensitive information to the new program.
 - Creating a file descriptor with the flag `O_CLOEXEC` marks the file
   descriptor to be closed when a new program gets executed. This
   automatically prevents file-descriptor leaks.
 - File descriptors should always be created with `O_CLOEXEC` and the flag
   should be cleared explictly before executing a new program if required.

If you like this blog post about the basics of Unix file I/O, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

[medium:ofinjch]:           http://medium.com/@fun_cuddles/opening-files-in-node-js-considered-harmful-d7de566d499f
[posix:close]:              http://pubs.opengroup.org/onlinepubs/9699919799/functions/close.html
[posix:dup]:                http://pubs.opengroup.org/onlinepubs/9699919799/functions/dup.html
[posix:execl]:              http://pubs.opengroup.org/onlinepubs/9699919799/functions/execl.html
[posix:fork]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html
[posix:fstat]:              http://pubs.opengroup.org/onlinepubs/9699919799/functions/fstat.html
[posix:open]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/open.html
[post:20170728]:            {% post_url 2017-07-28-data-structures-of-unix-file-io %}
