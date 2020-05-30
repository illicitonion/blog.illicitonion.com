+++
title = "Python, driving you to do the right thing, sometimes"

aliases = ["/2012/08/python-driving-you-to-do-right-thing.html"]
+++

This week I've been writing my first real Python. I mean, I've hacked together 100-line scripts before, but I've been writing real code from scratch, structured, with tests and everything.

I've noticed some really nice things, and some really horrible things.

**Nice thing**: Python doesn't let you hash mutable collections:

```python
>>> hash(set())  # mutable
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'set'
>>> hash(frozenset())  # immutable
133156838395276

>>> hash([1,2,3])  # mutable
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
>>> hash((1,2,3))  # immutable
2528502973977326415
```

This is great, because if you hash something, it tends to be in order to store it in a set or dict or something, and if you change the thing after hashing it, you've broken the contract of the set/dict.

Awesome. Guiding you toward doing the right thing.

**Horrible thing**: `unittest`[^1] has a method: `assertRaises`. Great - a simple, concise way of asserting that a single call raises an exception.

Except I wanted to assert that something is raised, without specifying what, so I tried skipping the exception type from the call to `assertRaises`.

```python
>>> import unittest
>>>
>>> def does_raise():
...  raise Exception()
...
>>> def does_not_raise():
...  return 1
...
>>> class TC(unittest.TestCase):
...  def test_raises(self):
...    self.assertRaises(Exception, does_raise)
...  def test_does_not_raise(self):
...    self.assertRaises(does_not_raise)
...
>>> unittest.main()
..
----------------------------------------------------------------------
Ran 2 tests in 0.002s

OK
```

WAIT! I called `self.assertRaises` with a method which _does not raise_! It's in the name and everything! It feels like if the first arg to `assertRaises` is a callable, and not a type, `unittest` could perhaps at least warn, if not throw.

If I hadn't run this test before implementing the backing code to see it fail, and seen it passed, I would have blindly been assuming that my code correctly raised an exception (as my test showed!) when it didn't!


[^1]: `unittest` from Python 2.7, backported as `unittest2` before
