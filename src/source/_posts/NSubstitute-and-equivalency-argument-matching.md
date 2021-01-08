---
title: NSubstitute and equivalency argument matching
date: 2021-01-06 12:15:06
tags:
  - nsubstitute
  - nsubstitute.equivalency
  - fluentassertions
---

When using the otherwise superb [NSubstitute](https://nsubstitute.github.io/), one thing has been bothering me for a long time: you cannot match arguments by equivalency.

To illustrate what I mean, let's use an example. Say you got the following interface

```csharp
public interface ISomeInterface
{
    void Use(Person person);
}

public class Person
{
    public DateTime Birthday { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

and a method that does something with it (for the sake of the example):

```csharp
static void DoSomethingWith(ISomeInterface service)
{
    service.Use(new Person
    {
        FirstName = "John",
        LastName = "Doe",
        Birthday = new DateTime(1968, 6, 1)
    });
}
```

Now how would you test that method? You'd start with the following:

```csharp
var service = Substitute.For<ISomeInterface>();
DoSomethingWith(service);
```

But now you'd like to validate that `service.Use()` was called with the right `Person`. Because NSubstitute has no built-in equivalency argument matcher, you would have to do this:

```csharp
service.Received().Use(Arg.Is<Person>(p => p.FirstName=="John" && p.LastName=="Doe" && p.Birthday==new DateTime(1968, 6, 1));
```

but obviously that is rather tedious and becomes unmanageable with more properties in `Person`. 

Alternatively, you can do:

```csharp
var service = Substitute.For<ISomeInterface>();
Person received = null;
service.WhenForAnyArgs(s => s.Use(null)).Do(ci => received = ci.Arg<Person>());

DoSomethingWith(service);

received.Should()
        .BeEquivalentTo(new Person
        {
            FirstName = "John",
            LastName = "Doe",
            Birthday = new DateTime(1968, 6, 1)
        });
```

This approach uses [FluentAssertions](https://fluentassertions.com/) and is a lot more robust, but it doesn't make for easy reading.


### Solution
Ideally, one would like to just write:

```csharp
var service = Substitute.For<ISomeInterface>();
DoSomethingWith(service);
service.Received()
    .Use(ArgEx.IsEquivalentTo(new Person
    {
        FirstName = "John",
        LastName = "Doe",
        Birthday = new DateTime(1968, 6, 1)
    }));
```

And if you install [NSubstitute.Equivalency](https://github.com/ModernRonin/NSubstitute.Equivalency), a little helper library I wrote for this purpose, you can do just that.

I called the type for accessing this matcher `ArgEx` to avoid creating import conflicts with the standard NSubstitute `Arg` type.
