---
layout: post
title: 'Git crimes - forgery'
short: "An odd view of git internals"
date: '2020-07-09 19:12:00 -0300'
---

`bash --version`: GNU bash, version 4.4.20(1)-release (x86_64-pc-linux-gnu)  
`git --version`: git version 2.17.1  
`INT (Arcana)`: DC 20  
`QOTD`:  
> Again, normally you’d never actually do this by hand.  
> _[Documentation of git core](https://git-scm.com/docs/gitcore-tutorial)_

Oh, you don't know me...

Git is nowadays the de facto standard for source versioning, and everyone who
has been in contact with code professionally has most likely used it, or at
least seen it. It really does the job, and its quite simple to use once the
basic concepts are understood.

Many would also have noticed the `.git` directory, an unmistakable sing for a
git repository. That's the place where git stores all the actual data (and
metadata) of your repositories.

Even a fresh new repository has this directory created from square one:

```
$ git init repo
Initialized empty Git repository in /home/bash-junkie/repo/.git/

$ cd repo

$ ls -al
bash-junkie@ubuntu-bionic:~/repo$ ls -lA
total 4
drwxrwxr-x 7 bash-junkie bash-junkie 4096 Jul  9 04:17 .git

bash-junkie@ubuntu-bionic:~/repo$ ls -lA .git
total 32
-rw-rw-r-- 1 bash-junkie bash-junkie   23 Jul  9 04:17 HEAD
drwxrwxr-x 2 bash-junkie bash-junkie 4096 Jul  9 04:17 branches
-rw-rw-r-- 1 bash-junkie bash-junkie   92 Jul  9 04:17 config
-rw-rw-r-- 1 bash-junkie bash-junkie   73 Jul  9 04:17 description
drwxrwxr-x 2 bash-junkie bash-junkie 4096 Jul  9 04:17 hooks
drwxrwxr-x 2 bash-junkie bash-junkie 4096 Jul  9 04:17 info
drwxrwxr-x 4 bash-junkie bash-junkie 4096 Jul  9 04:17 objects
drwxrwxr-x 4 bash-junkie bash-junkie 4096 Jul  9 04:17 refs
```

At a simple glance you may recognize some of the files and directories under
`.git`. The _config_ file is where local configuration for this particular
repository is stored. Places like _branches_ and _refs_, as well as the _HEAD_
file hold the pointers to every branch (and remote branches) the repo knows of,
and so on.

Of course, all the git commands we use handle this directory for us, and we
don't need to even know how it works. But what if...?

## A crash course on git internals

Almost everything in git is stored as objects, or as references to objects.
There are three (main) kinds of objects: **blobs, trees and commits**. All
objects, regardless of their type, have an unique id which consists of an
hexadecimal string of length 40 (adding up to 20 bytes if represented in
binary instead of hexadecimal).

You may have already came upon these ids when dealing with commits: the
"commit hash" is exactly the id of the commit object. And as the name
suggests, **an object's id is computed as a hash of the object's contents**.

Every object then has some contents, a type and an id computed from them as a
hash. All objects are stored under `.git/object` directory, in a organized
fashion. For example: the object with id
`5d9951bca047ccda20e7b2a6075d49a6eec0a521` will be stored in the file
`.git/object/5d/9951bca047ccda20e7b2a6075d49a6eec0a521`.

Notice how the first two characters of the hex string are used as a directory
inside `.git/object`, so all objects that start with the same two
characters will be grouped together in the same directory.

**Blobs are objects that hold contents of a single file**, a specific version
of it that is tracked by your repo. This means that every file in your git
repository will have a blob object associated with it for each time it was
changed (and committed).

**Tree objects** are the equivalent to blobs, but for **directories**. Inside the
contents of a tree object, you will find references to other trees and/or blobs,
which one for every file or directory that the tree contains.

Finally, a commit is an object which holds a reference to _a single tree_,
which is the root directory of your repository for the committed version; as well
as _one or more_ references to other commits objects as its parents. It also
contains other meta-data like commit's author, message and timestamp.

### Checking the objects

There is a git _plumbing command_ called `git cat-file` which can be used to
read the contents of any object. Using the `-t` flag will tell you the object's
type, and the `-p` flag will print its contents in a human-readable way.

For example, if we add a single file and commit to a freshly created repository:

```
$ echo "Hello git" >hi
$ git add hi; git commit -m "First commit"
[master (root-commit) 9fca3ba] First commit
 1 file changed, 1 insertion(+)
 create mode 100644 hi
```

We can start start inspecting first the commit. We don't need to specify the
whole 40 characters of the name of an object to use `cat-file`, as long as we
provide enough bits to uniquely identify it. In this case, the generated
commit was `9fca3ba`.

```
$ git cat-file -t 9fca3ba
commit

$ git cat-file -p 9fca3ba
tree 8c26cf2337ff9c9ac3ba1dea36436cb721f2ca9e
author Bash Junkie <junkie@ba.sh> 1594316223 +0000
committer Bash Junkie <junkie@ba.sh> 1594316223 +0000

First commit
```

As you can see from there, the object generated by git is a commit, and
its contents reference the working tree in the index at the moment of the commit,
and also holds all the metadata like the message we put there.

Note that there is no parent commit because it is the very first commit of
the repo. If we generate another commit on top of this one, we will see a
reference to the parent.

```
$ echo "Example line" >>hi

$ git commit -am "Another commit"
[master ca15ffc] Another commit
 1 file changed, 1 insertion(+)

$ git cat-file -p ca15ffc
tree 6412fa36e9b0f07fde2a8ba3b77cf8d91a248f53
parent 9fca3baef89171f24a061e3faccd4357498fc25a
author Bash Junkie <junkie@ba.sh> 1594316653 +0000
committer Bash Junkie <junkie@ba.sh> 1594316653 +0000

Another commit
```

Since `git cat-file` works with objects, we can use it to inspect the
contents of the trees and blobs too! For example, in the first commit:

```
$ git cat-file -t 8c26cf2337ff9c9ac3ba1dea36436cb721f2ca9e
tree

$ git cat-file -p 8c26cf2337ff9c9ac3ba1dea36436cb721f2ca9e
100644 blob 0dec2239efc0bbfabe4078f5357705ca93b5475e	hi
```

You can see the references to other blobs (and possibly trees), pairing git
object's ids with file names and permissions (in this case matching
`0dec22` to the name `hi` with permissions `100644`).

We can go one step further and check the contents of the blob, which should
match with our original _hi_ file.

```
$ git cat-file -t 0dec2239efc0bbfabe4078f5357705ca93b5475e
blob

$ git cat-file -p 0dec2239efc0bbfabe4078f5357705ca93b5475e
Hello git
```

As there is a blob for each _version_ of the file, if we do the same for the
second commit we should see different objects, but with similar contents.

```
$ git cat-file -p 6412fa36e9b0f07fde2a8ba3b77cf8d91a248f53
100644 blob 27c9f8894b64f86a17a7005a75c01b4940d22526	hi

$ git cat-file -p 27c9f8894b64f86a17a7005a75c01b4940d22526
Hello git
Example line
```

It is important to notice that blobs and trees do not hold any information on
author, timestamp or other metadata. They just have the contents of
the files and directories. This means that if you do the exact same thing in
_your_ repo, you should see **the same object ids** for trees and blobs. Try it!

For commits, however, ids may vary because the committer name and a timestamp
is embedded in the object and it gets mixed up in the hashing and calculation
of the id making it vary from repo to repo.

All these objects we've been looking at are in the `.git/object` directory.
Since we only did a few operations, these are the only objects there: both commits,
which hold different trees and different blobs.

```
$ tree .git/objects
.git/objects
├── 0d
│   └── ec2239efc0bbfabe4078f5357705ca93b5475e
├── 27
│   └── c9f8894b64f86a17a7005a75c01b4940d22526
├── 64
│   └── 12fa36e9b0f07fde2a8ba3b77cf8d91a248f53
├── 8c
│   └── 26cf2337ff9c9ac3ba1dea36436cb721f2ca9e
├── 9f
│   └── ca3baef89171f24a061e3faccd4357498fc25a
├── ca
│   └── 15ffc077d18d4f913aee8d68f8cd7444f74005
├── info
└── pack

8 directories, 6 files
```

Of course, if a file or directory does not change from one commit to another,
it's object (blob or tree) will get reused. But since **objects are
immutable** if any file changes, a new object will be created.

We can do a _empty_ commit to check how the tree get's reused:

```
$ git commit --allow-empty -m "Empty commit"
[master dbcf39f] Empty commit

$ git cat-file -p dbcf39f
tree 6412fa36e9b0f07fde2a8ba3b77cf8d91a248f53
parent ca15ffc077d18d4f913aee8d68f8cd7444f74005
author Bash Junkie <junkie@ba.sh> 1594317475 +0000
committer Bash Junkie <junkie@ba.sh> 1594317475 +0000

Empty commit
```

As it can be seen, parent commit has changed but the tree got reused; thus
saving us some space.

That covers most of the basics regarding git objects, but what about
branches?

Branches are stored each in a file with its name, and hold only a single
reference to the commit they point to. They are not stored as object, but
under the `refs` and `branches` directories.

```
$ cat .git/refs/heads/master
dbcf39f7fddd97df4d90a75bb52f41c9161adaea
```

Tags fall somewhere between the two: regular tags are just as branches, and
saved as references to commits under `.git/refs/tags`. Annotated tags are
another kind of object, and they hold not only a reference to a commit, but
also some metadata (the "annotated" part).

We are not going to pay much attention to how are branches stored here, we'll
be focusing con commits and objects.

# Creating a commit, the hard way

All git has to work with is the `.git` directory, there is no extra metadata
hidden elsewhere. So whenever we ask for our changes to be committed, git
takes care of updating the objects in a meaningful way to accomplish what
we need. So I was left wondering... would I be able to create a new commit
from scratch only by tampering with `.git` directory and its objects?

The [gitcore-tutorial](https://git-scm.com/docs/gitcore-tutorial) has an
excellent recompilation of git low-level commands, and explains how git
looks under the hood on a regular workflow. But it still uses git commands,
and I wanted to _really_ create a commit from scratch, and in a way, to trick
git.

Let's give it a try.

## Reverse engineering the objects

First thing naturally would be to take a look at how are these object stored
in memory. A first glance reveals that `git cat-file` is not an innocent
`cat`, but it does some sort of transformation: objects in disk appears as
garbled bytes of non-printable characters.

```
$ git cat-file -p 0dec2239efc0bbfabe4078f5357705ca93b5475e
Hello git

$ hexdump -c .git/objects/0d/ec2239efc0bbfabe4078f5357705ca93b5475e
0000000   x 001   K   �   �   O   R   0   4   `   �   H   �   �   �   W
0000010   H   �   ,   � 002  \0   5 001 005 203
000001a
```

The actual bytes stored in disk bare little resemblance to the actual
contents of the blob object. If git was using some kind of home-brewed
bitcode to encode its objects, there would be no way around finding the
documentation of that bitcode and actually implementing an encoder.

Hopefully, that was not the case as it turns out that objects are compressed
using [zlib](https://zlib.net/). This is not the most common way of
compression, but it can be dealt with relatively easily by using a capable
tool (in this case `pigz`).

```
$ file .git/objects/0d/ec2239efc0bbfabe4078f5357705ca93b5475e
.git/objects/0d/ec2239efc0bbfabe4078f5357705ca93b5475e: zlib compressed data

$ pigz -d <.git/objects/0d/ec2239efc0bbfabe4078f5357705ca93b5475e
blob 10Hello git
```

That is already starting to look better. It looks like git is indicating the
type of the object at the beginning, followed by what appears to be the length
of the blob, and finally its contents.

> The compression zlib does is not the same as the `.gz` mainstream way of
> compressing. So tools like `gzip` cannot handle it directly.
> However there are other tools that can deal with it. I found that `pigz`
> (`apt install pigz`) was the easiest one to use, but there are other clever
> ways to do it:
> * Openssl can be compiled and built with [zlib compression](https://github.com/openssl/openssl/blob/master/INSTALL.md#zlib-flags)
> but it does not come out of the box with that usually.
> * The [Python Standard Library](https://docs.python.org/3/library/zlib.html) has zlib among its
> options for encoding (`import zlib`).
> * And maybe my favorite for its
> cleverness is to trick gzip into uncompressing the data by injecting a header
> with gzip magic.
> ```
> echo -ne '1f8b080000000000' | xxd -r -p | cat - file | gzip -dc
> ```

We can better see the exact contents of the blob object by doing an hexudmp.
This will reveal that there are some invisible characters laying around. We
have another blob to use too, as another point of reference.

```
$ pigz -d <.git/objects/0d/ec2239efc0bbfabe4078f5357705ca93b5475e | hexdump -c
0000000   b   l   o   b       1   0  \0   H   e   l   l   o       g   i
0000010   t  \n
0000012

$ pigz -d <.git/objects/27/c9f8894b64f86a17a7005a75c01b4940d22526 | hexdump -c
0000000   b   l   o   b       2   3  \0   H   e   l   l   o       g   i
0000010   t  \n   E   x   a   m   p   l   e       l   i   n   e  \n
000001f
```

From this we can gather that the format of a blob is:
* The word `blob`, in plain text ascii
* A space character (0x20)
* The _size_ in bytes of the contents of the file. Weirdly, in base 10 and written in ascii. No encoding of the size here.
* A null string terminator (0x00, `\0` character)
* The actual contents of the blob

And those get compressed with zlib and placed under the objects directory. Right.
But how are the blob ids computed?

Well, for that we can experiment with different algorithms for hashing, both
the contents before and after compression and check which matches. Or we
could also check the documentation.

One way or another we would find out that git uses SHA1 algorithm for
getting object's id, and that the hash is performed **before the compression**.
To validate this with our friend, blob `0dec2239`, the `shasum` command comes in
handy, and it even has the SHA1 algorithm as its default one.

```
$ pigz -d <.git/objects/0d/ec2239efc0bbfabe4078f5357705ca93b5475e >some-blob

$ shasum some-blob
0dec2239efc0bbfabe4078f5357705ca93b5475e  some-blob
```

The hash id matches exactly, as we expect. We are now **blobbing experts**.

### Forging a blob

Remember the objective here, we want to manually ~forge~ create a commit only
by adding files to `.git/objects`, and make git believe a new commit exists.
For that we need a change, and to introduce a change we need a blob. Let's
use our new knowledge and forge one!

Let's add a new file, called `forged`, and have `Legit file\n` be its
contents. This would be the equivalent of doing `echo "Legit file" >forged`
and committing the new file. But we wont create the file at all, we will only add the blob's object.

Our _forged_ file has a length of 11 bytes (don't forget the end of line!),
so the blob will end up looking like `blob 11\0Legit file\n`. That's easy to
do.

```
$ echo -ne "blob 11\0Legit file\n" > forged-blob

$ hexdump -c forged-blob
0000000   b   l   o   b       1   1  \0   L   e   g   i   t       f   i
0000010   l   e  \n
0000013
```

A note here, we should not worry about the name (`forged`) we want our
file to have in the actual commit, because that is part of the tree. We'll
get there, but for now let's add this blob to the objects directory.

Recalling from the previous section, we first find out how our blob is going
to be called (by computing it's SHA1), and then we compress, rename and move
it to the proper sub-directory. We may need to create one for this.

```
$ shasum forged-blob
f3523e1b381ab0287b48121e833908d9cf23e3ba  forged-blob

$ pigz -zc <forged-blob >f3523e1b381ab0287b48121e833908d9cf23e3ba

$ hexdump f3523e1b381ab0287b48121e833908d9cf23e3ba
0000000 5e78 ca4b 4fc9 3052 6434 49f0 cf4d 512c
0000010 cb48 49cc 02e5 3b00 053e 00e1
000001b
```

`f3523e`, what a lovely name!

```
$ mkdir -p .git/objects/f3
$ mv f3523e1b381ab0287b48121e833908d9cf23e3ba .git/objects/f3/523e1b381ab0287b48121e833908d9cf23e3ba
```

Ok, everything is in place. Let's use `cat-file` on our new blob and see how
well we did. If everything worked out fine we should have git read our blob
without any issue.

```
$ git cat-file -t f3523e1b381ab0287b48121e833908d9cf23e3ba
blob

$ git cat-file -p f3523e1b381ab0287b48121e833908d9cf23e3ba
Legit file
```

Success! We forged a blob!!

It is of no use for us now, since it is not referenced anywhere yet, but the
fact that git read it tells us that we are on track. We need to plant a tree
that contains that blob now.

> It is also possible, of course, to get it wrong. These are some errors that git will output
> if it figures that some of the objects are malformed in some way.
> ```
> $ git cat-file -p 1d28276c78ad4e93dc2b1e5b082e6378d919f26d
> error: unable to parse 1d28276c78ad4e93dc2b1e5b082e6378d919f26d header
> fatal: Not a valid object name 1d28276c78ad4e93dc2b1e5b082e6378d919f26d
> 
> $ git cat-file -p ca523e1b381ab0287b48121e833908d9cf23e3b4
> fatal: invalid object type
> ```
> 
> There may be many more. See if you can catch them all!

## Climbing a tree

Rinse and repeat, let's do the same for a tree object. Let's grab one of the
trees we looked at before, decompress and dump its contents.

```
$ git cat-file -p 8c26cf2337ff9c9ac3ba1dea36436cb721f2ca9e
100644 blob 0dec2239efc0bbfabe4078f5357705ca93b5475e	hi

$ pigz -d <.git/objects/8c/26cf2337ff9c9ac3ba1dea36436cb721f2ca9e | hexdump -c
0000000   t   r   e   e       3   0  \0   1   0   0   6   4   4       h
0000010   i  \0  \r   �   "   9   �   �   �   �   �   @   x   �   5   w
0000020 005   � 223   �   G   ^
0000026
```

Uncompressing actually reveals some data as we expected, but this one is
making less sense than the blob. We can see the `tree` prefix, followed by
what we can assume it's the length of the tree's contents (from counting
bytes). But after that it becomes weird.

The permissions of the files are there, followed by the file name and a
string terminator: `100644 hi\0`. What follows _must_ be the blob's id, but
it is not ascii encoded... It is actually binary encoded!

```
$ pigz -d <.git/objects/8c/26cf2337ff9c9ac3ba1dea36436cb721f2ca9e | hexdump -C
00000000  74 72 65 65 20 33 30 00  31 30 30 36 34 34 20 68  |tree 30.100644 h|
00000010  69 00 0d ec 22 39 ef c0  bb fa be 40 78 f5 35 77  |i..."9.....@x.5w|
00000020  05 ca 93 b5 47 5e                                 |....G^|
00000026
```

Hopefully you can see how the bytes of the blob's id are there: they start in
the third byte of the second line of the output, `0d ec 22` and so on. It
matches the entire blob's id, up until the very last `5e`. Ja! That is
something we can deal with.

However, this example does not show how multiple entries are encoded, since we only
have one file in our tree. But the next one, from another test repository, will
help clarify that.

```
$ git cat-file -p 8988da15d077d4829fc51d8544c097def6644dbb
100644 blob f24c74a2e500f5ee1332c86b94199f52b1d1d962	example
100644 blob 557db03de997c86a4a028e1ebd3a1ceb225be238	hello
`
$ pigz -d <.git/objects/89/88da15d077d4829fc51d8544c097def6644dbb | hexdump -C
00000000  74 72 65 65 20 36 38 00  31 30 30 36 34 34 20 65  |tree 68.100644 e|
00000010  78 61 6d 70 6c 65 00 f2  4c 74 a2 e5 00 f5 ee 13  |xample..Lt......|
00000020  32 c8 6b 94 19 9f 52 b1  d1 d9 62 31 30 30 36 34  |2.k...R...b10064|
00000030  34 20 68 65 6c 6c 6f 00  55 7d b0 3d e9 97 c8 6a  |4 hello.U}.=...j|
00000040  4a 02 8e 1e bd 3a 1c eb  22 5b e2 38              |J....:.."[.8|
0000004c
```

Take a look at how the second block entry starts right next to the end of the
first one, no delimiter and no `\0`. The first bit of `100644` is directly
after the last byte of the _example_ blob's id (third line, sixth byte
right-to-left).

From what we just say, we reckon that the format of a tree object must be as
follows:
* The word `tree`, in plain text ascii, just as for the blob
* A space character (0x20)
* The _size_ in bytes of the contents of the tree entries (i.e. everything below). This is also in ascii, base 10.
* A null character (0x00)
* For each entry in the tree:
  * 6 ascii digits for the entry's permissions (e.g. `100644`)
  * A space character
  * The name of the entry (i.e. file/directory name)
  * A null character
  * 20 bytes of the blob or tree object's id for the entry, in binary

### A note on hash ids

What we are doing here is fine method to understand git internals and _add_
new commits on top of what the repo already has. It wont be useful to _alter_
objects already referenced.

The reason for this is that commits and trees have references to other
objects, and those references get hashed as part of the id of the commit or
tree. This means tampering with an already existing object means that its id
will change. Which in turn means that all commits and trees that reference it
will _also_ have to change, which makes the ids of those commits and trees to
also change... and so on.

Changing _any_ of the object could potentially mean having to rewrite all the
objects.

## Planting the tree

We now know how a tree file is encoded, we can forge one for our fake commit.
We want to add a new file to the existing tree, so we need the previous one
for the reference to the _hi_ file blob. Recalling from before:

```
$ git cat-file -p 6412fa36e9b0f07fde2a8ba3b77cf8d91a248f53
100644 blob 27c9f8894b64f86a17a7005a75c01b4940d22526	hi
```

So we need an entry that looks just like that one, and another one that has
same permissions, file name of `forged` and blob id of
`f3523e1b381ab0287b48121e833908d9cf23e3ba`, the one we forged before.

The tricky part here is how to convert the hex string into bytes. We can use
`xxd` another tool for hex dumping in "reverse mode" for that. Here is how
that looks like:

```
$ echo -n "f3523e1b381ab0287b48121e833908d9cf23e3ba" | xxd -r -p | hexdump -C
00000000  f3 52 3e 1b 38 1a b0 28  7b 48 12 1e 83 39 08 d9  |.R>.8..({H...9..|
00000010  cf 23 e3 ba                                       |.#..|
00000014
```

The `-r` flag just runs the whole thing in reverse, while the `-p` just tells
it to print without extra formatting bits (like line numbers and such).

Now we have to put it all together. Let's create the `forged-tree` file and
one by one append to it the entries. We can then figure out the header, and
ship it.

```
$ touch forged-tree

# First entry "100644 blob 27c9f8894b64f86a17a7005a75c01b4940d22526 hi"
$ echo -ne "100644 hi\0" >>forged-tree
$ echo -n "27c9f8894b64f86a17a7005a75c01b4940d22526" | xxd -r -p >>forged-tree

# Second entry "100644 blob f3523e1b381ab0287b48121e833908d9cf23e3ba forged"
$ echo -ne "100644 forged\0" >>forged-tree
$ echo -n "f3523e1b381ab0287b48121e833908d9cf23e3ba" | xxd -r -p >>forged-tree
```

Ok, both entries are in place, let's see how it is looking before we add the
header.

```
$ hexdump -C forged-tree
00000000  31 30 30 36 34 34 20 68  69 00 27 c9 f8 89 4b 64  |100644 hi.'...Kd|
00000010  f8 6a 17 a7 00 5a 75 c0  1b 49 40 d2 25 26 31 30  |.j...Zu..I@.%&10|
00000020  30 36 34 34 20 66 6f 72  67 65 64 00 f3 52 3e 1b  |0644 forged..R>.|
00000030  38 1a b0 28 7b 48 12 1e  83 39 08 d9 cf 23 e3 ba  |8..({H...9...#..|
00000040
```

So far so good. For the header, we count the bytes (we have 64, which is
nice) and then prepend it to the whole thing.

```
$ wc -c forged-tree
64 forged-tree

$ echo -ne "tree 64\0" | cat - forged-tree >forged-tree-with-header
$ hexdump -C forged-tree-with-header
00000000  74 72 65 65 20 36 34 00  31 30 30 36 34 34 20 68  |tree 64.100644 h|
00000010  69 00 27 c9 f8 89 4b 64  f8 6a 17 a7 00 5a 75 c0  |i.'...Kd.j...Zu.|
00000020  1b 49 40 d2 25 26 31 30  30 36 34 34 20 66 6f 72  |.I@.%&100644 for|
00000030  67 65 64 00 f3 52 3e 1b  38 1a b0 28 7b 48 12 1e  |ged..R>.8..({H..|
00000040  83 39 08 d9 cf 23 e3 ba                           |.9...#..|
00000048
```

And now we should be done. The process for placing it in the objects
directory should be the same that we used for the blob: compute the id,
compress and move.

```
$ shasum forged-tree-with-header
98fc72a299afc69bd6a2a2c2644516a34e7b7a66  forged-tree-with-header

$ pigz -zc <forged-tree-with-header >98fc72a299afc69bd6a2a2c2644516a34e7b7a66

$ mkdir -p .git/objects/98
$ mv 98fc72a299afc69bd6a2a2c2644516a34e7b7a66 .git/objects/98/fc72a299afc69bd6a2a2c2644516a34e7b7a66
```

Let's see if it worked, fingers crossed.

```
$ git cat-file -t 98fc72a299afc69bd6a2a2c2644516a34e7b7a66
tree

$ git cat-file -p 98fc72a299afc69bd6a2a2c2644516a34e7b7a66
100644 blob 27c9f8894b64f86a17a7005a75c01b4940d22526	hi
100644 blob f3523e1b381ab0287b48121e833908d9cf23e3ba	forged
```

Amazing!! We successfully planted our forged tree. We can even use one of the
_plumbing_ commands to compare it with the tree in the latest commit to check
for the differences.

```
$ git diff-tree -p 6412fa36e9b0f07fde2a8ba3b77cf8d91a248f53 98fc72a299afc69bd6a2a2c2644516a34e7b7a66
diff --git a/forged b/forged
new file mode 100644
index 0000000..f3523e1
--- /dev/null
+++ b/forged 
@@ -0,0 +1 @@
+Legit file
```

Look how the diff tells us that we added a file! There is something
reassuring about this that I could not explain. We took more than an hour
probably to get to this point, to do less that what git does with a single
command. But it has been worth it so far, if you ask me.

The forged tree, such as our previous blob, has no references, and is not
part of any commit.

We can fix that.

## Committing to the forgery crime

Rinse and repeat, we already know the trade. Figure out the format of the
commit object, then forge one. Easy.

Let's read the contents of our latest commit to get the feel for it's
contents.

```
$ git cat-file -t dbcf39f7fddd97df4d90a75bb52f41c9161adaea
commit

$ pigz -d <.git/objects/db/cf39f7fddd97df4d90a75bb52f41c9161adaea
commit 213tree 6412fa36e9b0f07fde2a8ba3b77cf8d91a248f53
parent ca15ffc077d18d4f913aee8d68f8cd7444f74005
author Bash Junkie <junkie@ba.sh> 1594317475 +0000
committer Bash Junkie <junkie@ba.sh> 1594317475 +0000

Empty commit
```

Ok, that was kind of unexpected. No weird characters? No encoding?

Indeed, it just appears to be plain text, apart from the usual `\0` and the
header. Let's double check with an hexdump.

```
$ pigz -d <.git/objects/db/cf39f7fddd97df4d90a75bb52f41c9161adaea | hexdump -C
00000000  63 6f 6d 6d 69 74 20 32  31 33 00 74 72 65 65 20  |commit 213.tree |
00000010  36 34 31 32 66 61 33 36  65 39 62 30 66 30 37 66  |6412fa36e9b0f07f|
00000020  64 65 32 61 38 62 61 33  62 37 37 63 66 38 64 39  |de2a8ba3b77cf8d9|
00000030  31 61 32 34 38 66 35 33  0a 70 61 72 65 6e 74 20  |1a248f53.parent |
00000040  63 61 31 35 66 66 63 30  37 37 64 31 38 64 34 66  |ca15ffc077d18d4f|
00000050  39 31 33 61 65 65 38 64  36 38 66 38 63 64 37 34  |913aee8d68f8cd74|
00000060  34 34 66 37 34 30 30 35  0a 61 75 74 68 6f 72 20  |44f74005.author |
00000070  42 61 73 68 20 4a 75 6e  6b 69 65 20 3c 6a 75 6e  |Bash Junkie <jun|
00000080  6b 69 65 40 62 61 2e 73  68 3e 20 31 35 39 34 33  |kie@ba.sh> 15943|
00000090  31 37 34 37 35 20 2b 30  30 30 30 0a 63 6f 6d 6d  |17475 +0000.comm|
000000a0  69 74 74 65 72 20 42 61  73 68 20 4a 75 6e 6b 69  |itter Bash Junki|
000000b0  65 20 3c 6a 75 6e 6b 69  65 40 62 61 2e 73 68 3e  |e <junkie@ba.sh>|
000000c0  20 31 35 39 34 33 31 37  34 37 35 20 2b 30 30 30  | 1594317475 +000|
000000d0  30 0a 0a 45 6d 70 74 79  20 63 6f 6d 6d 69 74 0a  |0..Empty commit.|
000000e0
```

And... no, that seems to be the case, which makes things really easy for us.

This is going to be the base of our commit object. Just copy the one for the
commit we already saw, and change a couple of values. We can tinker with the
timestamp to make it look like it came from the distant past.

```
$ cat >forged-commit <<EOF
> tree 98fc72a299afc69bd6a2a2c2644516a34e7b7a66
> parent dbcf39f7fddd97df4d90a75bb52f41c9161adaea
> author Bash Junkie <junkie@ba.sh> 14317475 +0000
> committer Bash Junkie <junkie@ba.sh> 14317475 +0000
>
> Forged commit
> EOF
```

Also, notice how the tree reference points to our forged tree, and the parent
is our current (master) commit. This will put it next in the chain.

Repeating the thing we did for the tree, we compute the length, add the
header; followed by the usual SHA1 for the id, compression and placement.

```
$ wc -c forged-commit
210 forged-commit

$ echo -ne "commit 210\0" | cat - forged-commit >forged-commit-with-header

$ shasum forged-commit-with-header
4effa5a21f066c87fc88be4ec13f93efae4509f7  forged-commit-with-header

$ pigz -zc <forged-commit-with-header >4effa5a21f066c87fc88be4ec13f93efae4509f7
$ mkdir -p .git/objects/4e
$ mv 4effa5a21f066c87fc88be4ec13f93efae4509f7 .git/objects/4e/ffa5a21f066c87fc88be4ec13f93efae4509f7
```

`4effa5`, again, a lovely name.

We have our commit placed, but we did not change any branch, so it wont be
accessible by means of a reference. We can still check it using
`cat-file`.

```
$ git cat-file -p 4effa5a21f066c87fc88be4ec13f93efae4509f7
tree 98fc72a299afc69bd6a2a2c2644516a34e7b7a66
parent dbcf39f7fddd97df4d90a75bb52f41c9161adaea
author Bash Junkie <junkie@ba.sh> 14317475 +0000
committer Bash Junkie <junkie@ba.sh> 14317475 +0000

Forged commit
```

This is incredible, we have built from scratch a brand new commit. We now can
do everything we would do with any other regular commit. Just keep in mind
that it is not referenced by any branch, so it can be potentially pruned if
you tell git to do so (it's a dangling commit).

Just for the sake of it, we can do a `git show` on this commit and see what
we get.

```
$ git show 4effa5a21f066c87fc88be4ec13f93efae4509f7
commit 4effa5a21f066c87fc88be4ec13f93efae4509f7
Author: Bash Junkie <junkie@ba.sh>
Date:   Mon Jun 15 17:04:35 1970 +0000

    Forged commit

diff --git a/forged b/forged
new file mode 100644
index 0000000..f3523e1
--- /dev/null
+++ b/forged
@@ -0,0 +1 @@
+Legit file
```

Look at the file! Look at the date! It is everything we expected to see!

Checking out to that commit will instantly make the file `forged` file appear,
even though we never actually created it.

There are some subtleties with the index and the references that may need to
be deal with, but overall we manage to create a new commit from the ground
up, bit by bit.

Thank you! and congratulations if you made it this far! I encourage you to play around
with this and forge your own objects in git. It's real fun.
