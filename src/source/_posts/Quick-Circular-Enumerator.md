---
title: Quick Circular Buffer
date: 2020-10-08 22:20:08
tags:
  - data structures
---

Recently, I found myself for the first time in my 25+ year career needing infinite iteration over a finite set of elements. 

The context was generation of examples. For certain data types, there is a finite set of constant example values provided, but the actual number of values that will be
required are not known beforehand.

Interestingly, I could not find a nuget for this, so I had to build it myself. Maybe someone else will need it, too, within the next 25 years, so here it goes:

```csharp
public class SimpleCircularContainer<T> : IEnumerable<T>

   readonly T[] _values;
   public SimpleCircularContainer(params T[] values) => _values = values;
   public IEnumerator<T> GetEnumerator() => new Enumerator(_values);
   IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
   class Enumerator : IEnumerator<T>
   {
       readonly T[] _values;
       int _nextIndex = -1;
       public Enumerator(T[] values) => _values = values;
       public bool MoveNext()
       {
           _nextIndex++;
           if (_nextIndex >= _values.Length) _nextIndex = 0;
           return true;
       }
       public void Reset()
       {
           _nextIndex = 0;
       }
       public T Current => _values[_nextIndex];
       object IEnumerator.Current => Current;
       public void Dispose() { }
   }

```

Usage is like this:

```csharp
[TestFixture]
public class SimpleCircularContainerTests
{
    [Test]
    public void Enumeration_wraps_around()
    {
        var underTest = new SimpleCircularContainer<string>("alpha", "bravo", "charlie");
        underTest.Take(10)
            .Should()
            .Equal("alpha", "bravo", "charlie", "alpha", "bravo", "charlie", "alpha", "bravo", "charlie",
                "alpha");
    }
}
```