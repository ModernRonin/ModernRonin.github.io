---
title: >-
  FluentAssertions: How to assert collection's elements with equivalence AND
  order
date: 2020-10-12 22:30:08
tags:
  - FluentAssertions
  - TDD
---

I've been using the marvellous [FluentAssertions](https://fluentassertions.com/) library for many years, yet I still discover useful details I had not been aware of.

Whenever you want to check returned collections/`IEnumerable<>`s and the elements don't have compare-by-value semantics (so in the vast majority of cases at least in my experience), you need to use `.Should().BeEquivalentTo()` rather than `.Should().Equal()`.

However, there is a catch: `BeEquivalentTo()` does not check for equal order of elements. So, for the past few years, I used to work around this in more or less complicated ways - until today to my delight I noticed that `BeEquivalentTo()` has an overload with a configuration parameter that lets you specify exactly what I need:

```csharp
underTest.Should().BeEquivalentTo(expected, cfg => cfg.WithStrictOrdering());
```

The only, minor drawback is that, naturally, this signature doesn't allow `params` style passing of the elements to compare against. 
