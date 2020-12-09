---
title: Anonymous types and FluentAssertions
date: 2020-08-27 14:18:55
tags:
  - FluentAssertions
  - TDD
---

Say you are testing the return value of a method returning an interface:
```csharp
public class UnderTest
{
    IInfo GetInfo() 
    {
        // ...
    }
}


interface IInfo 
{
    string Text { get; };
    int Number { get; }
}
```

How to comfortably assert on the return value if you don't know the concrete type? It turns out FluentAssertions handles anonymous types perfectly:

```csharp
var underTest= new UnderTest();
underTest.GetInfo().ShouldBeEquivalentTo(new { Text="bla", Number= 13});
```

