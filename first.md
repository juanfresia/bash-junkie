# Bash expansions

Arcana check DC:
Recommended doses of coffee:

Oh, bash expansions. You've heard of them. You've use them. You've seen them
everywhere: from the one-liners for invoking your command line tools, to those
convoluted scripts no one really understand..

The concept is really easy to understand: you got a variable `FOO`, you
want to get it's contents, then just prepend a `$` and you are done. But things
beyond that start getting confusing. Should I use `$FOO` or `${FOO}`? What about
the double quotes: is it better `"${FOO}"` or just `${FOO}`?

Let's not even get into those weird ones like `"${@}"`, really intuitive indeed.
My personal favourite is this one `"${FOO%%:*}"`. I mean, what is that even **supposed to do**?

So, let's try to make some sense from all of that. At the end, these things are
not hard at all, they are just **really confusing** and **annoyingly
unintuitive**.

## All is a string

Variables in bash are not typed, you can't declare a _bool_ or a _int_ variable.
Instead all variables are treated equal: as strings. This is why it is easy in
bash to operate with strings and regexes, but it is really hard to do a simple
addition.





## 
