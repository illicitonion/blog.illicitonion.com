+++
title = "Extending LINQ Distinct to take Funcs"
+++

I've recently been getting to know C#, and one thing that struck me as very nice is how elegant LINQ is. Most of the time. Every now and then, I come across a corner which just makes me think "this could be a little nicer!" - occasionally these are syntax issues which would require a change to the parser itself[^1], but thanks to extension methods, a lot of things are achievable.

Something that in particular frustrated me today was that I had to write a few `IEqualityComparer<T>` classes; in couple of places I wanted to select distinct elements from an `IEnumerable<T>`, where "distinct" was specialised depending on context, rather than based in the elements' `IEquatable<T>` implementation.

My first guess, was that I could write something along the lines of:

```cs
MyList.Distinct((x, y) => x.Property == y.Property)
```

But, surprisingly, the only implementations of `IEnumerable<T>.Distinct` either take no arguments (using the Default `EqualityComparer<T>`, which defers to `Object.Equals`), or take an `IEqualityComparer<T>`. Having to define a whole new class which implements `IEqualityComparer<T>`, just to filter a list, though on the face of it sane, doesn't seem very LINQy to me. And worse than that, because C# doesn't support Java's inline overloads[^2], this needs to be a properly fleshed out class!

Well, I follow the old rule of "Do something once. Do it again. That third time, make something do it for you", so I made something generate that class for me. As I said before, my aim was to be able to type:

```cs
MyList.Distinct((x, y) => x.Property == y.Property)
```

to filter a list. Something, however, is missing here. The `IEqualityComparer<T>` interface has two methods on it: `public bool Equals(T x, T y)` and `public int GetHashCode(T obj)`. The compiler certainly can't (in the general case) infer what `HashCode` I mean from my equality function, but then, it shouldn't really need to get the `HashCode` anyway, right? I hoped I could get away without implementing `GetHashCode`, but...

`Distinct` evaluates lazily. That's useful - if you have an infinite list, it means that calling `Distinct` will only do any processing when you actually want the next element, rather than running forever on the infinite list. It does this by keeping a `Set<T>` of all of the elements it's returned to you so far. When you request the next element, it gets the next element, and checks whether it's in the set. If it's in the set, it's been returned before, so is discarded and the next element tried. The check for whether it's in the set uses the `HashCode`.

Because the jump from "I have an arbitrary function for equality" to "Generate me a function which generates unique numbers based on that equality" is non-trivial, my dreams of:

```cs
MyList.Distinct((x, y) => x.Property == y.Property)
```

are gone, but we can still improve on the status quo:

```cs
MyList.Distinct((x, y) => x.Property == y.Property, x => x.Property.GetHashCode())
```

is still a lot more LINQy than having to define classes. In fact, for this example, as long as `Property` is `IEquatable`, we can make it much tidier:

```cs
MyList.Distinct(x => x.Property)
```

Here we have the extension methods and helper code which actually implements these two forms of `Distinct`:

```cs
public static class EnumerableExtensions
{
  public static IEnumerable<TSource> Distinct<TSource>(
    this IEnumerable<TSource> source,
    Func<TSource, TSource, bool> comparator,
    Func<TSource, int> hasher
  ) {
    return source.Distinct(new FuncComparer<TSource>(comparator, hasher));
  }

  public static IEnumerable<TSource> Distinct<TSource, TResult>(
    this IEnumerable<TSource> source,
    Func<TSource, TResult> uniqueProperty
  ) where TResult : IEquatable<TResult> {
    return source.Distinct(new FuncComparer<TSource>(
      (x, y) => uniqueProperty(x).Equals(uniqueProperty(y)), x => uniqueProperty(x).GetHashCode())
    );
  }

  class FuncComparer<T> : IEqualityComparer<T>
  {
    private readonly Func<T, T, bool> m_Comparator;
    private readonly Func<T, int> m_Hasher;

    public FuncComparer(Func<T, T, bool> comparator, Func<T, int> hasher)
    {
      m_Comparator = comparator;
      m_Hasher = hasher;
    }

    public bool Equals(T x, T y)
    {
      return m_Comparator(x, y);
    }

    public int GetHashCode(T obj)
    {
      return m_Hasher(obj);
    }
  }
}
```

And here it is in use:

First a boring class which we can actually compare:

```cs
public class Property : IEquatable<Property>
{
  public Property(string x)
  {
    X = x;
  }
  public string X { get; private set; }

  public bool Equals(Property other)
  {
    return other != null && Equals(X, other.X);
  }

  public override int GetHashCode()
  {
    return X == null ? 0 : X.GetHashCode();
  }
}

public class HasProperty
{
  public HasProperty(Property property)
  {
    Property = property;
  }
  public Property Property { get; private set; }
}
```

and now some tests:

```cs
[Test]
public static void DistinctShouldFilterByUniqueProperty()
{
  var unfilteredArray = new[] { Get("abc"), Get("def"), Get("abc"), Get("gij") };
  var expectedArray = new[] { Get("abc"), Get("def"), Get("gij") };

  var filteredArray = unfilteredArray.Distinct(x => x.Property).ToArray();

  Assert.AreEqual(expectedArray.Length, filteredArray.Length);
  foreach (var pair in filteredArray.Zip(expectedArray, (actual, expected) => new { Actual = actual, Expected = expected }))
  {
    Assert.True(pair.Expected.Property.Equals(pair.Actual.Property));
  }
}

[Test]
public static void DistinctShouldFilterByCustomComparer()
{
  var unfilteredArray = new[] { Get("abc"), Get("def"), Get("abc"), Get("gij") };
  var expectedArray = new[] { Get("abc"), Get("def"), Get("gij") };

  var filteredArray = unfilteredArray.Distinct((x, y) => x.Property.Equals(y.Property), x => x.Property.GetHashCode()).ToArray();

  Assert.AreEqual(expectedArray.Length, filteredArray.Length);
  foreach (var pair in filteredArray.Zip(expectedArray, (actual, expected) => new { Actual = actual, Expected = expected }))
  {
    Assert.True(pair.Expected.Property.Equals(pair.Actual.Property));
  }
}
```

If you're looking really closely, you may have noticed that I've used `.Equals` rather than `==`. The multitude of equality comparisons in most languages, including C#, is really rather annoying; at least if we specify that `TResult` is an `IEquatable`, we can vaguely trust `.Equals`...



[^1]: As an example, wouldn't it be so much nicer to be able to say:
```cs
foreach (x, y) in (xs, ys) {
  Console.WriteLine(x);
  Console.WriteLine(y);
}
```

rather than

```cs
foreach (var pair in xs.Zip(ys, (x, y) => new {X = x, Y = y}))
{
  Console.WriteLine(pair.X);
  Console.WriteLine(pair.Y);
}
```

But hey, this isn't Python!

[^2]: In Java, you can instantiate implementations of interfaces inline, roughly:
```cs
MyList.Distinct(
  new IEqualityComparer<T> {
    public bool equals(T x, T y) {
      return x.Property == y.Property;
    }
  });
```

a bit more verbose than my ideal, but certainly better than having to write out a whole class!
