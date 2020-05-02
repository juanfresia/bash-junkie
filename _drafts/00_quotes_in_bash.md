---
layout: post
title: 'Quoting in bash'
short: "To quote or not to quote"
date: '2020-04-23 23:25:00 -0300'
---

# Quoting variables in bash
`bash --version`: GNU bash, version 4.3.48(1)-release (x86_64-pc-linux-gnu)  
`INT (Arcana)`: DC 15  
`QOTD`:  
> We will see how to deal with those cases, its all a matter of quoting.  
> _Bash Junkie, 2019_

Bash variables can be tricky, their behaviour is not the same as variables in
other programming languages (the ones we are most likely to be familiar with).
When you ask bash to recall the value of a variable, some weird things may happen,
if you are not careful.

Say we want to count 

```bash:

```

To start with, their contents are always strings, no matter what. Defining a
variable in a shell or a script is easy, we just use the assignment operator: `=`.

Double quotes are optional, but needed for strings with special characters, such
as spaces. Note that variable expansion and substitutions works as expected
during the assignment, and that using single quotes will prevent any of that
from happening.

Variables could also be multi-line! But that usually makes things a lot harder
to deal with, so avoid them. Sometimes there is no option, because we want to
capture the multi-line output of a program, for example `cat some-file.txt`.

> We will see how to deal with those cases, its all a matter of quoting.
> _Bash Junkie, 2019_

Snippet 1: defining some variables
```
FOO=hello
BAR=
LONG_TEXT="quotes for long strings"
FOO2="$FOO world. Expansion works!"
FOO3='$FOO world. Unless you use single quotes'
CMD=$(cat some-file.txt)
```
