---
layout: post
title: 'How to /procrastinate'
short: "Proclaimed `/proc` procedures."
date: '2020-05-01 16:10:00 -0300'
---

`bash --version`: GNU bash, version 4.4.20(1)-release (x86_64-pc-linux-gnu)  
`uname -a`: Linux ubuntu-bionic 4.15.0-96-generic #97-Ubuntu SMP Wed Apr 1 03:25:46 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux  
`INT (Arcana)`: DC 10  
`QOTD`:  
> Proclamation: procure to procrastinate on `/proc`'s procryptic processes and proceed to procreate procephalics.

From time to time I lose myself to the narrow alleys and hidden passages of
the `/proc` directory. Each time I go in I try to find something I had not
seen before, maybe something that was added recently, or perhaps a specific
file whose meaning I did not understand in the past and now it adds another piece
to the puzzle.

It's fascinating to go in searching for a specific piece of information and
founding it, laying there. Or even better, finding something one _was not
looking for_.

Maybe this resonates with you, maybe it doesn't. But we all procrastinate at
some point and I more than once done it with the `/proc` directory.

## /proc's procryptic processes

What is exactly that `/proc` directory, anyway?

Well, most people are introduced to it by the verse "it is a _special_
directory that holds some information about processes". And that is a fair
statement. It is indeed _special_, and it holds information about processes.
You can check by doing a quick listing of its contents to see a lot of
_numbered subdirectories_ appear in front of your eyes.

Each of them corresponds to the process with the same _pid_.

```
$ ls -1 /proc
1
10
1012
105
1090
[and many, many more not shown here]
```

Enter any of those directories and you will find a whole lot of information
about the process it is related to. Some of the files are pretty obvious, but
others are quite cryptic and more obscure.

```
$ ps
  PID TTY          TIME CMD
 2402 pts/0    00:00:00 bash
 8728 pts/0    00:00:00 ps
$ ls -1 /proc/2402
attr
autogroup
auxv
cgroup
clear_refs
cmdline
comm
coredump_filter
[again, shortened, but by all means try it yourself!]
$ cat /proc/2402/cmdline
bash
```

Check out how we used the `cmdline` file inside our shell process' directory
under `/proc` to figure out how what was the command that executed it (that's
the meaning of `cmdline` file here). Why would I need that for, you ask yourself?

Well, for this particular file, not much... the information is available
already through the ps command. But, there is also a lot of information
hidden here waiting to be found.

### A window to the kernel

The `/proc` directory is not actually part of your real filesystem, those
files are actually not in your disk at all. It is a _pseudo-filesystem_, yet
another illusion created by the kernel to give us a peek into it's internal
data structures through the interface of a filesystem. This follows one of
unix principles: everything is a file.

In fact, the `/proc` directory _itself_ has nothing special on its own. We
can turn _any_ directory we want in our filesystem into a window to the
kernel. We simply use the `mount` command to instruct the kernel to give us
"a copy" of the `proc` _pseudo file-system_ (sometimes called _procfs_) and
mount it into our filesystem tree.

```
# Create an empty dir
$ mkdir my-proc

# Request (kindly) to the kernel to mount the pseudo-filesystem
# procfs into ./my-proc
$ sudo mount -t proc proc ./my-proc

# Profit
$ ls -1 my-proc/
1
10
1012
```

The `mount` system call accepts a parameter which specifies the _type_ of
mount to perform. There are a lot of types supported, some of them refer to
_device_ mounts (the "real" ones, those that are backed up with a hard drive
for example), and other to _non device_ mounts (the "fake" ones, like `proc`
or in-memory temporary filesystems).

> The second `proc` in `mount -t proc proc /dir` is actually a quirk from the
> mount command that requires you to specify a device: `mount -t [type]
> [device] [target]`. When using the `proc` type, _anything_ can actually be
> used as the device, the kernel will know you are trying to mount the _pseudo
> filesystem_.

It is typical for procfs to be mounted automatically by the system under
`/proc` directory. But of course one does never know when it may need it in
other places... okay, okay, you'll probably never need to do that, this is just
for sport.

It is worth noting that this pseudo filesystem is read only. If we try to
write in it, or create directories or files inside it we will get a rather
confusing error:

`mkdir: cannot create directory ‘/proc/somedir’: No such file or directory`

No such file?? That's exactly why I am trying to create one!!

However, there are exceptions. Some even _more special_ files inside `/proc`
directory, like the ones in  `/proc/sys` subdirectory, can actually be modified.
Writing to these files provides a way of communicating with the kernel and
changing some of its configuration. The kernel will intercept the write and
do some stuff in reaction to the new value.

The kernel is watching you, some files may return different values depending
on which permissions you have. Some files only reveal its contents to a user
with certain capabilities. On top of that, some files are tied to namespaces,
so only will show you stuff that is in the same namespace as you are (pids
are a great example of this).

## Landmarks of `/proc`

There is _a lot_ of information under `proc` pseudo filesystem. I'll only
show the ones that I found more interesting, but refer to the `proc`
manual page for a detailed list an explanation of every single one of the
files there. But let me warn'ya: its a one way journey.

#### Regular process stuff

Of course, one of the most useful things under `proc` is the plethora of
information on all the processes running in the system (well, technically is
all process of a a particular pid namespace). You can find data of a particular
process inside `/proc/[pid]`, where _pid_ is the ID of the process.

In fact, the `ps` utility we mentioned earlier uses the `/proc` directory to
gather all the information. You can test this by removing the `/proc`
mountpoint and attempting to use `ps`: it will kindly require you to mount
proc back again in `/proc`. Try to figure out which other unix tools rely on
`/proc` to be mounted!!

```
# Override /proc by mounting /home on top :)
$ sudo mount --bind /home /proc

# Oops
$ ps
Error, do this: mount -t proc proc /proc
```

> Fun bonus fact here: `/proc` filesystem is fixed to the pid namespace of the
> process that executed the mount. So when creating a new pid namespace it is
> important to also get rid of the old `/proc` mount, otherwise the process
> will be able to peek outside its namespace. This is particularly relevant to
> container implementation.

Just to go over some of the information under `/proc/[pid]`
here is a quick list of the ones I liked the most:

* `/proc/[pid]/mounts`: list of filesystem mounts in the process mount
namespace. Check its contents and guess who reads from here.
* `/proc/[pid]/root`: this is interesting, a symlink to the process
"root directory" and it reflects how the process _itself_ sees its root.
So if it has mounted some extra stuff, you will be able to see those mounts
here (`proc` man pages have a really nice demo on this).
* `/proc/[pid]/fd`: directory containing symlinks to all open files, indexed by
file descriptor (`lsof` relies on this). Also, `/proc/[pid]/fdinfo` gives us
access to some metadata of the file descriptor table (seek position in the
file, access with which the file was open, etc)
* `/proc/[pid]/oom_score`: how "close" is the process
to being targeted by the Out Of Memory Killer. A high number indicates the
process have more chances of being sacrificed to free up some resources.
This only makes sense if some memory limits have been set to the process by a cgroup.
* `/proc/[pid]/maps`: show the regions of memory currently mapped in the
process virtual memory space. Also gives us information of shared libraries
that are mapped onto process memory.

And there are a whole lot more covering threads (tasks), namespaces, general
status, memory and io consumption, even the process very own _personality_?.
The list is very large and I encourage you to check it in the man pages.

#### That huge file

If you check the size of the files under `/proc` you may have noticed that almost
all of them have a size of 0. Which makes sense, because they are "fake" files.
However, there is one that goes the opposite way:

```
$ ls -alh /proc/kcore
-r-------- 1 root root 128T May  2 05:54 /proc/kcore
```

128TB of data? That's a lot. Of course, it is not a real file as we already
know. If you ask for it's contents the kernel will generate it for you on the
go. It is that huge because it represent the whole kernel virtual address
space, in ELF format and in real time. It can, in theory, be used to debug
the kernel with gdb.

In the same lines, not every address is mapped to RAM; some of that address
space is wired to other devices. You can check which portions of the virtual
address space are actually mapped to io devices in `/proc/iomem`.

#### Filesystems types known by mount

This is a specific system-wide bit of information just to go full circle.
Remember when we talk about the `-t` flag of mount command, and how it
supported several types of filesystem? Well, it all depends on which features
your kernel has, and which _kernel modules_ you have to handle filesystem
formats.

There is a handy file called `/proc/filesystems` that tells you which types
of filesystem your kernel currently supports. You can even see which ones
require a block device (the "real" ones) and which are purely virtual (the
"fakes", tagged with `nodev` in the file)

## Much, much more

Well, its 4am and we only scratched the surface here. There is a lot of more
things un `/proc` that we have yet to see. It is in part because we are
peeking into the kernel, and the linux kernel for sure is not simple at all!!

I found it fascinating, and hope you do so too and find in the examples I picked
enough curiosity to try and explore it on your own. Go, my friend, and get
yourself lost under `/proc` pseudo filesystem.






