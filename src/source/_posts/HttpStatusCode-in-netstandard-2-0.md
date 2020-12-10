---
title: HttpStatusCode in netstandard 2.0
date: 2020-09-28 22:04:01
tags:
  - netstandard2.0
  - BCL
  - gotchas
  - C# language
  - F# language
  - language design
---

Today I stumbled across a strange omission in netstandard2.0: the `HttpStatusCode` enum is incomplete (as opposed to, say, the netcoreapp3.1 version). For example, it does not contain a label for 422 (Unprocessable Entity). 

Fortunately, because C#'s enums are really just wrappers around `int`s and thus not all that type-safe, you can just do

```csharp
var code= (HttpStatusCode) 422;
```

to work-around this omission.

While C#'s enum implementation is handy in this situation, in general I'm not a big fan of it. Like C++'s enums, it really brings almost no value to the table when compared to a static class with constants, except that you can more easily parse names to values. 

I'd much rather have F#'s union types in C# than enums, but looking at the latest additions to the language, specifically patterns and records, it is probably only a matter of time until union types are implemented to. 
