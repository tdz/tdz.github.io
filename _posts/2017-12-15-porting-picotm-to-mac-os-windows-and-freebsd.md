---
layout:     post
title:      Porting picotm to Mac OS, Windows and FreeBSD
date:       2017-12-15 12:30:00 +0100
tags:       [bsd, cygwin, freebsd, macos, picotm, porting, windows]
og-image:   share2.png
---

I spent the previous weeks on porting [picotm][picotm] from Linux to a
number of other operating systems. Ports to Mac OS X, Windows and
FreeBSD will be available in picotm's next release. in this blog post
we go through them one by one and look at the major points and pitfalls.

<!-- excerpt -->

In it's current release, picotm, the system-level transaction manager,
only runs on Linux systems. Supporting more operating systems was high
on my *TODO* list for quite a while. During the previous weeks I finally
got to work on ports to Mac OS, Windows and FreeBSD.

Picotm provides transactional semantics (i.e., automatic locking and
error handling) for a large set of low-level functionality, such as

 -  memory operations (transactional memory),
 -  C memory and string functions (concurrent `mem*()` and `str*()` calls
    on shared memory),
 -  memory allocation (`malloc()` and `free()` on shared memory),
 -  file-descriptor I/O (concurrent reading and writing within the same file),
 -  process working directory,
 -  C math functions, and
 -  transactional data structures (shared lists, multisets, etc).

All this functionality is implemented as a set of shared libraries, built
on top of a single core library that manages all transactions in the
application. It's all C code.

The goal for each new port is to get the core transaction manager running
on the new system and then get (most of) the actual functionality working.

#### FreeBSD

The first port was from Linux to [FreeBSD][freebsd]. In terms of systems
programming, FreeBSD is very similar to Linux. It has the same or similar
tools and compilers. It's a [POSIX][posix]-compatible system and supports
most of the same programming interfaces that Linux uses. For interfaces
that are missing, there's usually a replacement with equivalent
functionality.

Getting FreeBSD was really easy. The FreeBSD project even provides a complete
pre-configured disk image for virtual machines.

The first thing to port was the build system. Picotm uses GNU autotools,
which are designed for portability among Unix systems. Many people love to
hate the autotools, but they are actually quite good. I resolved an
additional dependency on the `realpath` tool by changing the affected
build scripts to work without any additional tool, so this change benefits
the original Linux build scripts as well.

Next after the build system was the core transaction manager. This library
is written in C11 with dependencies on the operating system's mutexes,
condition variables and time sources. FreeBSD comes with a recent version
of clang, which provides a good C11 compiler. As a POSIX system, FreeBSD
provides all of the required mutex, condition variables and time interfaces.
So no problems here.

One major difficulty in porting picotm is the difference in supported
interfaces on each operating system. If a system provides a specific
interface, picotm tries to offer a transactional variant. If an interface
is *not* available, picotm does to *not* offer it. For example, on Linux
the GNU libc provides the non-standard function
[`rawmemchr()`][gnu:rawmemchr] for searching a character within a memory
buffer. Therefore picotm provides `rawmemchr_tx()`, a transactional
variant. The C standard library on FreeBSD does not provide `rawmemchr()`,
so picotm doesn't provide `rawmemchr_tx()`.

Supporting system-specific interfaces is easy with the GNU autotools. With
autoconf, the macro `AC_CHECK_DECL` adds a test for the declaration of a
specific function or macro in a specific header file. In the source code
one can test for the result and change the program's behaviour accordingly.

For example, the package configuration scripts contain the following
test for `rawmemchr()`.

```
AC_CHECK_DECL([rawmemchr],
              [AC_DEFINE([PICOTM_LIBC_HAVE_RAWMEMCHR],
                         [1],
                         [Define to one of you have rawmemchr.])],
              [],
              [@%:@include <string.h>])
```

This test searches for a declaration of `rawmemchr()` in the header file
`<string.h>`. If it find the declaration, it defines the C pre-processor
token `PICOTM_LIBC_HAVE_RAWMEMCHR` to `1`.

In the picotm source code, we can do a test and only provide a transactional
implementation if `PICOTM_LIBC_HAVE_RAWMEMCHR` is defined to `1`.

``` c
#if defined(PICOTM_LIBC_HAVE_RAWMEMCHR) && PICOTM_HAVE_RAWMEMCHR
void*
rawmemchar_tx(const void*s, int c)
{
    /* transactional variant of rawmemchr() */
}
#endif
```

This works fairly well and I use this scheme for *all* supported system
interfaces of the C standard and math libraries, and the POSIX thread
library. It's not complicated, just very tedious to do this for the several
hundred interfaces that are supported by picotm.

I had to add a few minor changes throughout picotm's source code to turn
non-portable C code into portable one, and that's it! After this, the test
cases succeeded on FreeBSD and I consider it supported.

#### Mac OS

The next system to support was Mac OS X. Like Linux and FreeBSD, it's a
Unix system and mostly compatible with the POSIX standard.

Because Mac OS only runs on Apply hardware, I bought an old Mac mini on Ebay.
Setting up the system for the first time was a bit frustrating. The problem
was that it's a model from 2007 and has long been discontinued. The final
version of Mac OS X that run on the device is 10.6.8, with development tools
being out of date and compilers not supporting C11.

Thanks to the excellent work of the [MacPorts][macports] project, a lot of the
current Unix software is available on Mac OS X. So I was able to install
recent releases of gcc and build tools even on this old Mac mini.

Even though Mac OS X is a POSIX system, it is more different from Linux than
FreeBSD, and required another set of changes and fixes.

For example, C11 provides static assertions that are evaluated at compile
time. Doing something like

``` c
static_assert(sizeof(int) == 4, "Values of type 'int' are not 4 bytes wide.");
```

tests if the size of an integer is 4 bytes. If the assertion evaluates to
`false`, the compiler stops the build.

The C standard library of the Mac OS's SDK (i.e., Xcode 3.6) does not support
static compile-time assertions. In this and other cases of missing functions,
I reimplemented the minimum functionality within picotm. The system's
implementation is prefered, but picotm can fall back to it's internal code
if necessary. For the missing `static_assert()` macro, there's now a macro of
the same name which looks like this.

``` c
#define __PICOTM_LIBC_STATIC_ASSERT(expr, stmt)                          \
    (__extension__({                                                     \
        int assertion_holds[1-(2*!(expr))] __attribute__((__unused__));  \
     }))

#ifndef static_assert
#define static_assert   __PICOTM_LIBC_STATIC_ASSERT
#endif
```

Note that the actual implementation is in `__PICOTM_LIBC_STATIC_ASSERT`. The
code only defines `static_assert` if it's not already defined by the system.

A similar problem happend with Pthread barriers. A
[barrier][wikipedia:barrier] is a data structure for synchronizing the
execution of multiple threads. It waits to be acquired by a specific number
of threads until the acquiring threads can pass. In picotm, it's used to
synchronize the threads of each test case.

The implementation of a barrier is a combination of a mutex and a condition
variable. All of them; mutices, condition variables and barriers; are provided
by POSIX threads, but barriers are optional. While the implementation of a
barrier is trivial, it's missing at least on this old version of the Mac OS
SDK. I had to implement the POSIX barrier interface within picotm, using the
system's POSIX mutices and condition variables.

Finally there was one major problem with time and clock interfaces. The POSIX
standard defines the function `clock_gettime()`, which retrieves a time stamp
from one of the system clocks. This interface is missing on Mac OS. I didn't
write a portable implementation within picotm, but instead changed the
function that acquired the time stamp in the first place. On Mac OS X, it
reads time stamps from non-portable interfaces, on all other systems it
reads time stamps using `clock_gettime`.

Because of the use of system-dependent interfaces for reading time stamps,
I had extend the error-handling interface as well. The Mac OS kernel returns
low-level error codes in the type `kern_return_t`. If a time stamp could not
be acquired from the system, this error code is now reported to the
transaction's error recovery phase, so that applications can act upon it.

With these changes and some additional minor fixes, the test cases
completed successfully on Mac OS. Another system supported!

#### Windows

Windows is *not* Unix, so any Unix-specific interface is not natively
supported. Fortunately, there is [Cygwin][cygwin] a POSIX interface
implemented on top of the Windows API.

This means that the Windows port does currently not support Windows' native
interfaces. For example, it's not possible to perform a transaction write
operation to a file using [`WriteFile_tx()`][msdn:writefile]. But using
Cygwin, it is possible to achieve the same result with
[`write_tx()`][posix:write].

With Cygwin and all the previous fixes for FreeeBSD and Mac OS, the port
itself was straight-forward. I only had to fix a few minor issues and the
test cases completed successfully within a few hours of work. I'm happy
to have Windows support on the feature list.

A few issues remain though. I would be interested in supporting Windows'
native interfaces, but it's a lot of work. At this point it doesn't make
sense for me to spend time on this unless there is real demand for these
interfaces.

The other issue is with dynamic linking support. Picotm comes as package of
multiple libraries that depend on each other. Linking a DLL requires all its
dependencies  (i.e., other DLLs) to be already installed. This makes linking
picotm libraries as DLLs complicated. It's not hard to fix, but the result is
rather ugly and therefore left-out; and dynamic linking on Windows
is disabled for now. Statically linking against picotm's libraries is no
problem.

#### Next Steps

The current ports lay the ground work for supporting even more operating
systems. Coming from FreeBSD and Mac OS, possible next targets are other
Unix operating systems, such as Android and OpenBSD.

A very interesting target is [FreeRTOS][freertos], an operating system for
low-power, single-board computers and the IoT. Running code transactionally
has the potential of significantly increasing the reliability of these
devices.

Then there are the more obscure targets, such as [FreeDOS][freedos] or
[AmigaOS][wikipedia:amigaos]. And of course, maybe one day I can port
picotm to [opsys][github:opsys], my own little operating-system project.


#### Summary

In this blog post, we've looked at the different ports that are supported
by the next release of picotm.

 - picotm is developed mainly on Linux, but supposed to be portable to
   a wide range of different operating systems.
 - FreeBSD is very similar to Linux, so mostly minor changes were required
   for a port.
 - Mac OS X is a Unix system like Linux and FreeBSD. It required the
   reimplementation of a few interfaces, but it's still mostly compatible.
 - Windows uses a different interface than Linux or any other Unix system.
   While the native interfaces require a significant amount of work to
   be supported, all of the actual functonality is supported by using
   the Cygwin compatibility layer.
 - With these operating systems supported, more can be added easily.

If you like this blog post, please subscribe to the RSS feed, follow on
Twitter or share on social networks.

[cygwin]:               http://www.cygwin.com/
[freebsd]:              http://www.freebsd.org/
[freedos]:              http://www.freedos.org/
[freertos]:             http://www.freertos.org/
[github:opsys]:         http://github.com/tdz/opsys
[gnu:rawmemchr]:        http://www.gnu.org/software/libc/manual/html_node/Search-Functions.html#index-rawmemchr
[macports]:             http://www.macports.org/
[msdn:writefile]:       http://msdn.microsoft.com/en-us/library/windows/desktop/aa365747(v=vs.85).aspx
[picotm]:               http://picotm.org/
[posix]:                http://pubs.opengroup.org/onlinepubs/9699919799/
[posix:write]:          http://pubs.opengroup.org/onlinepubs/9699919799/functions/write.html
[wikipedia:amigaos]:    http://en.wikipedia.org/wiki/AmigaOS
[wikipedia:barrier]:    http://en.wikipedia.org/wiki/Barrier_(computer_science)
