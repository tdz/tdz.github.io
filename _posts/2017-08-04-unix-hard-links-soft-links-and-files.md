---
layout:     post
title:      Unix Hard Links, Soft Links, and Files
date:       2017-08-04 10:00:00 +0200
tags:       [c, bsd, file, hardlink, howto, io, linux, open, posix, symlink, tutorial, unix]
og-image:   tux.png
---

This blog post describes the relation ship between file names and files on
Unix file systems. We'll also look at soft links and how all this interacts
with file descriptors.

<!-- excerpt -->

#### Creating a New File

Let's create a new file `my_file.txt` our application. We do this by
calling [`open()`][posix:open] with the file's name.

``` c
int fd0 = open("/home/joe_user/my_file.txt", O_RDWR|O_CREAT, S_IRUSR|S_IWUSR|S_IRGRP);
```

As we've seen in the previous [blog post][post:20170728] this creates a file
descriptor, an open file description, and a file buffer. You should read that
entry if you are interested in the relationship among file descriptors, open
file descriptions and file buffers.

![Data structures after opening `/home/joe_user/my_file.txt` once.]({{ site.baseurl }}/img/unix1.png)

In the file system, there's now a file name that refers to the file buffer
we just opened.

![File system after opening `/home/joe_user/my_file.txt`.]({{ site.baseurl }}/img/vfs1.png)

A file name that refers to a file buffer is also called *hard link.* There
can be multiple links to the same file buffer. Or in other words, the same
file buffer can be made available under different names.

#### Adding Hard Links

An additional name for a file is created with the command
[`link()`][posix:link]. The function takes an existing file name and a new
file name. If successful, the new name refers to the existing file name's
buffer.

Let's create an additional hard link of the name
`/home/joe_user/my_hard_link.txt`.

``` c
link("/home/joe_user/my_file.txt", "/home/joe_user/my_hard_link.txt");
```

Our file system now looks like this.

![File system after creating `/home/joe_user/my_hard_link.txt`.]({{ site.baseurl }}/img/vfs2.png)

What happens if we open the new file name from within our application?

``` c
int fd1 = open("/home/joe_user/my_hard_link.txt", O_RDWR|O_CREAT, S_IRUSR|S_IWUSR|S_IRGRP);
```

It's the same file buffer as before, so the result is the same as if
we'd opened the file by the original name.

![Data structures after opening `/home/joe_user/my_hard_link.txt` once.]({{ site.baseurl }}/img/unix2.png)

The application has opened two file descriptors, two open file descriptions,
and they both refer to the same file buffer.

This is also the reason why we cannot go through all files on a file
system to sum up how much space they consume on the disk. Since multiple
file names can refer to the same file buffer, we could calculate a cumulative
size that might be larger than the actually consumed size; maybe even
larger than the maximum size of the disk.

#### Unlinking, Closing, and the Removal of File Buffers

A call to [`unlink()`][posix:unlink] removes a file name from the file
system. Let's call `unlink()` on both file names and the file system will
be empty.

``` c
unlink("/home/joe_user/my_file.txt");
unlink("/home/joe_user/my_hard_link.txt");
```

Once the final link is gone, there's no way to create a new link to the
file buffer. But the application still has these files open! How does that
work?

For each file buffer, Unix maintains two counters: the link counter and
the open counter.

The *link counter* contains the number of links refering to a file buffer.
After being created, each file starts with a link count of *1.* When we
added a hard link with `link()` the counter went up by one. When we
removed a hard link with `unlink()` the counter went down by one.

The *open counter* countains the number of times the file buffer was opened,
or more specifically the number of open file descriptions refering to the
file buffer. When a file buffer is not opened, the open counter is *0.*
Each time we opened a file buffer with `open()` that counter went up by one.
Only when we later close the file descriptors and the open file
descriptions get removed, the counter goes down.

When both, link counter and open counter, reach zero at the same time
Unix removes the file buffer from the disk.

Let's close both file descriptors to remove the file buffer from the
disk.

``` c
close(fd0);
close(fd1);
```

#### Soft Links

Hard links have one major drawback, which is that they only work on the
file system that also maintains the buffer. So if you have one partition
for your computer's operating system, an another partition for your personal
data, it's not possible to create a hard link from one file system to the
other.

Unix file systems provide a solution called soft links, also called
symbolic links or symlinks. Although it has a file name, a soft link is
not a regular file, but a special file that stores a full file path.
When an application opens the soft links, it actually opens the file at
the path that is stored in the soft link. Because a file path is stored
(instead of a reference to a file buffer), it's independent from the
underlying file system.

Let's go back to the example of `/home/joe_user/my_file.txt`. We can create
a soft link with a call to [`symlink()`][posix:symlink]. Like `link()` before,
it takes the original file's path and the path of the symlink.

``` c
symlink("/home/joe_user/my_file.txt", "/home/joe_user/my_soft_link.txt");
```

With the soft link in place, the file system looks like this.

![File system after creating `/home/joe_user/my_soft_link.txt`.]({{ site.baseurl }}/img/vfs3.png)

We have a file name `my_file.txt`, which refers to a file buffer. And we
have a file name `my_soft_link.txt`, which refers to the soft-link file,
which in turn refers to the original file name.

The existance of a soft link does not have any impact on the actual file,
or that file buffers's link counter. So a file can be removed, even if it
is the target of a soft link. If this happens, the soft link goes stale
and trying to open it returns an error.

#### Summary

In this blog post, we went through a tour of how hard links and soft links
work on Unix, and how they affect the life time of file buffers.

 - Initially each file buffer is referenced by a file name.
 - The file name is called *hard link.*
 - There can be multiple hard links ot the same file buffer.
 - Opening any of a file buffer's hard links opens the file buffer.
 - A file buffer can not be opened any longer after all hard links have been
   removed.
 - A file buffer is removed from the disk after all hard links have been
   removed and the file buffer is not opened by any application.
 - Hard links don't work across file-system boundaries.
 - Soft links are a special type of file that reference other files by
   their path names.
 - Opening a soft link opens the referenced file.
 - Soft links can reference files on other file systems.

If you like this blog post about the basics of Unix File I/O, please
subscribe to the RSS feed, follow on Twitter or share on social networks.

#### Footnotes

[^1]:   There are a number of good books on Linux and Unix programming. If
        you're interested in Linux system programming I'd recommend Michael
        Kerrisk's [The Linux Programming Interface][man7:tlpi].

[posix:close]:              http://pubs.opengroup.org/onlinepubs/9699919799/functions/close.html
[posix:open]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/open.html
[posix:link]:               http://pubs.opengroup.org/onlinepubs/9699919799/functions/link.html
[posix:symlink]:            http://pubs.opengroup.org/onlinepubs/9699919799/functions/symlink.html
[posix:unlink]:             http://pubs.opengroup.org/onlinepubs/9699919799/functions/unlink.html
[post:20170728]:            {% post_url 2017-07-28-data-structures-of-unix-file-io %}
