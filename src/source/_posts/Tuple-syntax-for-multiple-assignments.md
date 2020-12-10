---
title: Tuple syntax for multiple assignments
date: 2020-09-2 21:17:13
tags:
  - C# language
  - Tuples
---

This is a short one. I just noticed how neat tuple syntax can be for multiple assignments:

```csharp
var (oldReference, newReference) = (searchPattern(oldName), searchPattern(newName));
```
