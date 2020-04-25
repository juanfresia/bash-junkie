---
layout: post
title: 'Bash conditionals'
date: '2020-04-23 23:25:00 -0300'
---

`bash --version`: GNU bash, version 4.3.48(1)-release (x86_64-pc-linux-gnu)  
`INT (Arcana)`: DC 12  
`QOTD`:
> Conditionals.  
> You know, the _if... then... else_ stuff.  
> Bash have them too!

## Bash has conditionals

Today I explored the lands of bash conditionals. Like most things in bash,
they are usually used "by recipe", without ever understanding how they really
work.

_"Of course I know how it works"_, I hear you say, _"I know conditionals!"_.
And that is fair, anyone familiarized with a programming languages (or even
English!) understands the basic structure of a conditional.

However, conditionals in most programming languages have a very **defined structure**:

```
if <condition> then
    <do stuff>
else
    <do another stuff>
end
```

We _understand_ that the "condition" that defines how the execution flows
has to be a boolean proposition, which means it has to evaluate to either be
true or false. We can put there a comparison between variables, a boolean
value, a call to a method that returns such as value.

In bash however we don't really have variables. Not in the same way as most
languages. We cant just simply equal two variables as part of the expression
and call it a day: bash is instead based around **commands**. Let's look at
an example, a first attempt to use bash conditional.

```bash
$ cat conditionals-1
#! /bin/bash

foo=1

if $foo == "1"; then
  echo hi
fi
$ ./conditionals-1
./conditionals-1: line 5: 1: command not found
```

Hmmm... it did not work. How bad is that? No matter how hard we try to fit
bash into our standard model for conditionals we cannot seem to have it to
work. The `command not found` error somehow makes matters a little bit more
off-putting. Here are some more failed attempts.

```bash
$ export foo=1
$ if $foo == 1; then echo hi; fi
1: command not found
$ if $foo == "1"; then echo hi; fi
1: command not found
$ if ( "$foo" == "1" ); then echo hi; fi
1: command not found
```

So we do what we all do: we search for a solution online and find the magic
of the _double brackets_ `[[]]`. We are done with it...

```bash
$ cat conditionals-1-revisited
#! /bin/bash

foo=1

if [[ $foo == "1" ]]; then
  echo hi
fi
$ ./conditionals-1-revisited
hi
```

...or are we?

What did actually change? It must be bash weird syntax, I said to myself the
first time I saw it. But then later on you see that replacing the double
brackets with simple brackets also works. Less characters, brilliant!

```bash
if [ $foo == "1" ]; then echo hi; fi
fi
```

Then you see some people doing the same thing you do, exactly the same...
except that for some reason they changed the `==` for a `-eq`. Weird right? I
mean, that it also works. So many ways to do the same thing.

```bash
if [ $foo -eq "1" ]; then echo hi; fi
fi
```

At that point its all good, until you stumble upon other "formats" of bash
conditionals and if you are me, you start asking questions. How does this
thing _really_ work?

```bash
if test $foo -eq 1; then
  echo hi
fi

if mountpoint /var/log; then
  <do stuff if /var/log is actually a mountpoint, duh!>
fi
```

Of course, it is enough to see a couple of examples to get a feeling on what
works and what doesn't. But sometimes is not all that clear.

## Everything is a command

So, we want to make sense of bash conditionals. The best place to start is, of
course, the documentation. Let's take a simplified version of what its on bash
manual:

```bash
if list; then list; fi
    The if list is executed. If its exit status is zero, the then list is executed.
```

Here _list_ just means as a command or a group of commands; separated by `|`,
`||`, `&&` or `;`. Note that it does not say anything about _expressions_ or
_conditions_. The "condition" part of the if statement in bash is **always**
of the same type as the "execution" bit, and that is a command.

Bash particularly cares about the **exit status** of commands, and that is
what it will use to determine if the _then list_ of commands gets
executed. We can start experimenting:

```bash
$ if true; then echo hi; fi
hi
$ if false; then echo hi; fi
```

Worth noting `true` is a command that always succeeds (exit code 0), and
`false` is a command that always fails (exit code 1).

> _We all have some day when we felt like unix false command:_  
> `false - do nothing, unsuccessfully`

Thit explains why the `mountpoint` conditional worked. It's just a command
and its exit codes indicates wether or not the directory is a mountpoint,
exactly the kind of stuff our if structure wants.

However, what if we want to compare the output of a program? Or if we have a
variable that we want to use as part of the condition? Well, as it is often
the answer in Unix school, _there is a tool that does exactly that, and only
that_.

## Comparison command

Let's suppose you want the very simple `if $foo == 1` conditional to work;
but you only know the base structure of bash conditionals (i.e. that
everything is a command), and some C. You are very ingenious too.

You realize you could write a program in C that takes as inputs some values
and operators and **changes its exit code** depending on the result of the
comparison. You could then make a call to your program as part of the _if
list_ of commands: chose the proper arguments you are done. Let's do that!

```c
// compare.c

#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char* argv[]) {
    // Check for number of arguments.
    // We need have some exit code, even if it makes no sense
    // because no comparison was made!
    if (argc != 4) {
        printf("You must provide exactly 3 arguments\n");
        exit(1);
    }

    // The input command should be: compare elem1 op elem2
    char* elem1 = argv[1];
    char* op = argv[2];
    char* elem2 = argv[3];

    int cmp = strcmp(elem1, elem2);

    if (strcmp(op, "==") == 0) {
        return cmp;
    }

    printf("invalid operator\n");
    exit(1);
}
```

That's a simple silly program that takes tree arguments, checks the middle
one is `==` and then just does a `strcmp` between the other two arguments.
Easy but powerful enough to cover our very first use case.

```bash
$ export foo="foo"
$ export bar="bar"
$ if ./compare $bar == "foo"; then echo hi; fi
$ if ./compare $foo == "bar"; then echo hi; fi
hi
```

Note that the exit code somehow encodes **both** the result of the comparison,
and the success or failure of our program. The _if statement_ can never
know if an exit code of `1` means "failure to execute" or "the comparison
turned out to be false". It's up to the person coding the comparison command
to define what's the default value to output, things should go wrong.

Now we got the basic, we can extend this program however we want, adding more
and more features: more input variables, logic operators, integration with
filesystem to check for files, etc. However, we won't have to, because it's
already done for us; or at least to a certain number of features.

### The `test` command

In the old times a command was developed to fulfill the very same requirement
that we explored with our `compare` program. That command was a binary called
`test`. Things have changed since then, but you can still find it in modern
distributions under `/usr/bin/test` (try it! it's part of the _coreutils_
package).

> I know what you are thinking. _Where is the source code?_  
> Well, it can be found [here][1]. You are welcome.

So, let's see how it works, shall we?

A quick look through its documentation shows us several valid expressions, and
a million of flags to do several things. Just to list a couple of them:

```bash
STRING1 = STRING2             # the strings are equal
STRING1 != STRING2            # the strings are not equal
INTEGER1 -eq INTEGER2         # INTEGER1 is equal to INTEGER2
INTEGER1 -le INTEGER2         # INTEGER1 is less than or equal to INTEGER2
-e FILE                       # FILE exists
EXPRESSION1 -o EXPRESSION2    # either EXPRESSION1 or EXPRESSION2 is true
```

We have much more than we asked for: we can do logic and/or operations
between expressions, compare strings and integers, and even check for several
properties of a file. Keep in mind that this is still a regular binary,
taking regular arguments and changing its return code.

Using the command we just introduced, we can rewrite our first example like
this. Of course we can always make sure the `test` command is in our `PATH`
to avoid using the full path `/usr/bin/test`.

```bash
$ export foo="foo"
$ export bar="bar"
$ if /usr/bin/test $bar = "foo"; then echo hi; fi
$ if /usr/bin/test $foo = "bar"; then echo hi; fi
hi
```

Why is not every bash conditional wrote using `/usr/bin/test`? What about the
`[]` and `[[]]` expressions that also seemed to work?

Well, some people back in the day decided that an if with a condition
enclosed in brackets looked better than one with the test command _[citation
needed]_. So they created _another_ command, and called it `[`.

No, it's not a mistake, `[` it's the actual name of the command!
Go, ask your machine _where is_ the `[` command
and it will tell you it's in `/usr/bin/[`.

The `[` command is exactly the same thing as the `test` command, the only
difference is that `[` always needs an extra argument at the end: the
matching closing `]`. If you check, they are even built from the same source
code! You can find this in `test.c` (linked above, see? You are welcome again).

```c
#if LBRACKET
# define PROGRAM_NAME "["
#else
# define PROGRAM_NAME "test"
#endif
```

That blew my mind the first time I heard of it. Anyway, that's part of the
mystery solved. We can rewrite any condition we used with `test`, to an
equivalent with `[` and it'll do the same thing.

Of course, using the whole path `/usr/bin[` kinda defeats the purpose.
Fortunately these commands are likely placed in your path already. This
leaves us with:

```bash
$ export foo="foo"
$ export bar="bar"
$ if [ $bar = "foo" ]; then echo hi; fi
$ if [ $foo = "bar" ]; then echo hi; fi
hi
```

A funny note on this `[` command is that it **allways** requires you to add
the `]` as the last argument. And it has to be a _separate_ argument. If not
you will get the infamous `missing ']'` error, even if the character is
there!

```bash
$ if [ $foo == foo]; then echo hi; fi
-bash: [: missing `]'
```

When you think about it as arguments for a command, its crystal clear.

### The built-in apotheosis

Those commands were being used **a lot** in shell scripting, so at some point
the shells themselves started porting the logic of `test` command as
_builtins_. A shell builtin is just a command that is part of the shell
binary. It's more portable because you don't depend on extra programs and
it's more performant because it does not need a new process to be executed.

Bash included both builtins `test` and `[`. The code probably drifted
slightly, but the functionality is similar for the most part. You can see,
under the `test` command man pages the following note:

```
NOTE: your shell may have its own version of test and/or [, which usually
supersedes the version described here. Please refer to your shell's
documentation for details about the options it supports.
```

That has been true for bash since a while now. When you run `test` or
`[`, it will actually run the builtin before even checking for the command.
You can still use the _coreutils_ version specifying the full path.

## The improved "test command"

So, we've come a long way already. We now understand most of bash conditional
expressions, but there is one thing left: the double brackets `[[]]`.

At this point I though just what you are thinking: _they must be another command
that was promoted to builtin_. In a way that is right, but technically it's wrong.
Let me explain what I mean.

Double brackets are a _first class citizens_ among bash syntax: `[[` is not a command, it is not a _builtin_ but it is a _keyword_. As such it has a special meaning inside bash, and has it's own rule for parsing.

> You can use the `type` builtin to check how bash will interpret a particular string:
> ```bash
> $ type [[
> [[ is a shell keyword
> $ type [
> [ is a shell builtin
> $ type test
> test is a shell builtin
> ```
> This has no practical use at all, but its a nice thing to know, right?

Bash interprets expressions starting with `[[` as _compound commands_. You
can get all the details in the manual pages, but in short it means that
patters with the format `[[ EXPRESSION ]]` are parsed as a whole unit by Bash
itself: it just takes the whole thing, evaluates it as a compound command and
makes it return a value depending on the result of the evaluation.

```bash
[[ expression ]]
      Return a status of 0 or 1 depending on the evaluation of the
      conditional expression expression.
```

So, now we should ask... what are the rules for that evaluation?

Well, the rules are called _conditional expressions_ and are roughly the same
as the ones that we already covered for `test` and `[` builtins. This means
the format of what goes inside can be the same as those commands. Whoa, that
was exciting. Where is the catch? Why not add another alias to the `test`
command, why do even care?

### The power of the parser

Yes, the general flavor and format of the conditional expression `[[`
supports is almost cloned from it's predecessors. Mostly to avoid driving
shell script programmers a new way of doing their ifs. That's not the interesting
part of this.

Having `[[` integrated so deeply in bash shell as to become part of it's very own
syntax means a couple of things:

We are no longer tied to the `command` format (which is also followed by
builtins). Characters with special meaning inside a command (such as `|` or
`&&`) had to be escaped to be used within a command. Ever wondered why the
`-a` to _and_ two conditions? 

Now that we can use `&&` and `||` for logical operations, `-a` and `-o` are
no longer supported. We can also use parenthesis for grouping and other
operators (like `<`) that previously needed escaping for having special
meaning inside a command.

Another key difference is that we no longer have to worry about empty
variables and wring substitutions. Essentially quotes are needed whenever we
want bash to _pass the command and empty argument_ if the variable is empty
or undefined. If we don't quote, the variable is omitted.

```bash
# Parenthesis and , sign need to be escaped
# -a and -o as logical operators
# Quoting is needed if variables can be empty
[ \( "$foo" -eq 1 -a "$bar" -eq 2 \) -o 3 \< 2 ]

# Quoting is not needed
# This will work even if foo or bar are empty!
[[ ( $foo == 1 && $bar == 2 ) || 3 < 2 ]]
```

But `[[` is a _compound command_, so it is not tied to that substitution. In
fact, it performs substitutions in a more friendly and orderly way. This
allow anther nice feature: _short circuiting_. With builtins this cannot be
`done, simple because all expansions have to be performed _before_ calling
the builtin. However, `[[` is part of the shell syntax, and it does short
circuiting!

```bash
# Second part of the -o could be ommited given that whole expression
# will always be true regardless
[ 1 -eq 1 -o $(touch oops; echo 1) -eq 1 ]

# Same here
[[ 1 == 1 || $(touch oops; echo 1) == 1 ]]
```

You can try it on your own, check both of them and see if any generates the
`oops` file. You will see that the `[[` one performs a more clever way of
evaluating the expression, and by doing so it can short circuit when possible.

Another _big_ difference lies in error handling. Remember the quirk commands
had when dealing with errors? A command has only its exit code to output both
the result of the comparison _and_ tell if an error happened; there is no way
around that. But making `[[` part of shell syntax gives us more power on how
to handle errors. Now the bash itself can halt the whole thing up if an error
happens while traversing the _compound command_, just the same as it would
stop a command from executing if a bad substitution happened.

Take the `-d` flag, that checks if a particular file is a directory. It of
course _always_ need a parameter (i.e. the file to check), but look what
happens when we omit that value with `[` and with `[[`.

```
# No error, it just evaluates to false
$ [ -d ]

$ [[ -d ]]
-bash: unexpected argument `]]' to conditional unary operator
-bash: syntax error near `]]
```

Those are the main differences and advantages `[[` has over `[`, but there is
one more thing worth noting. `[[` is part of bash, other shells (such as zsh)
also understand the keyword, but they may have slightly different syntax or
features. So _there is no standard_ for `[[`.

The old `[` (and `test`) both have a subset of features that are standarized
by POSIX, and most shells (even old ones like `[`) will have that subset for
sure. So when writing a script that has to work with the old world, it's not
advised to use `[[`.

## Putting it all together

### Challenges


[1]: https://github.com/coreutils/coreutils/blob/master/src/test.c "test source code from coreutils"