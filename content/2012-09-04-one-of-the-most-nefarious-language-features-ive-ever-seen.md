+++
title = "One of the most nefarious language features I've ever seen"
+++

Take a look at the following code:

`file1.py`:
```python
from package.lib import do_something_which_raises
from otherpackage.myexception import MyException
try:
  do_something_which_raises()
except MyException:
  print 'Oops, raised'
```

`file2.py`:
```python
from otherpackage.myexception import MyException
def do_something_which_raises():
  raise MyException()
```

What would you say if I told you that this code didn't print 'Oops, raised!', but instead bubbled `MyException` all the way up to terminate execution?

"Sure, you have two classes named `MyException`".

You can see that both files import exactly the same exception class from exactly the same package. I promise you, there is only one file named `myexception.py`, which contains exactly one class named `MyException`. If you walk all the entries of `sys.path`, you will only reach one definition of `MyException`.

"That's not possible!" I hear you say.

That's certainly what I thought. But I was faced with the fact that I'd moved a couple of files around in my source tree, fixed up the package references, and three quarters of my tests were failing. I had no bloody clue why.

It turns out, class equality in python depends not on whether two types were defined the same, but rather, whether they were loaded exactly the same.

You see, `file1.py` and `file2.py` are in different directories; the structure is actually:

```
./file1.py
./package/file2.py
./package/otherpackage/myexception.py
```

but both `top_level` and `top_level/package` are in my `sys.path`. `.` is always implicitly in `sys.path`, and I added `/path/to/package` to my `$PYTHONPATH`.

So when, in `file1`, I import from `otherpackage.myexception`, Python actually goes "`otherpackage`... Well, I don't have a file named that in the current directory... Or a folder named that... Let's start going through `$PYTHONPATH`".

For `file2`, it goes "Aha! The current directory has a folder named `otherpackage`! And inside it is a file named `myexception`! And that defines a class named `MyException`! You're sorted!"

But as far as Python knows, these files were reached differently. They have different paths to themselves (one is `/path/to/package / otherpackage/myexception.py` and the other is `./ otherpackage/myexception.py` - the fact that, `.` happens to be the same as `/path/to/package` is neither here nor there. Python's package cache, apparently, doesn't do that level of resolution[^1] (I guess it assumes no one would be silly enough as to have overlapping `sys.path` entries).

So when I said "you will only reach one definition of `MyException`", I was only telling half a truth. You will only reach one definition, but you will reach it twice. And that confuses python.

So here's a warning for you. Don't have overlapping `sys.path` entries. And if you do, always reference every definition therein from a consistent top-level folder. Because if you get this wrong, equality isn't equality, and everything goes to hell.



[^1]: This is conjecture; I've looked through the source of the equality functions, and exception definitions, in both 2.7 and 3.3 to see whether anything funky was going on there, I've not yet delved in to the package cache, but this seems like reasonable conjecture, I'll probably get around to reading through the package cache source at some point, and may follow up then with a blog post and/or patch to python.
