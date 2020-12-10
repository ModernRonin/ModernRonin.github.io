---
title: Extract a parameter object with R#
date: 2020-11-24 21:27:00
tags:
  - ReSharper
  - refactoring
---


At some point in the many years since I've been using R#, they have sneaked in a very useful feature, but hidden it so well that it took me until now to find it.


Say you have a horrible method like:
```csharp
Task UploadFileAsync(string fileName,
            IList<Resource> resources,
            ResourceCacheMethod cacheMethod,
            bool isThumbnail,
            IProgress<double> progress = null)
```

You put the cursor on/inside the method name, then press `Ctrl-Shift-R` (or whatever triggers `Refactor this...` for you), then select, on the very bottom of the context-menu, `Transform parameters`. In the dialog popping up you call the target class, for example, `FileUpload`, and, eh voila, the signature is transformed to

```csharp
Task UploadFileAsync(FileUpload fileUpload)
```

and a corresponding `FileUpload` type is created, and all calls are adjusted. 


My only quibble with this is that it *generates a constructor in `FileUpload` with all parameters*, which typically is not what you want when you refactor legacy code and need this kinda thing. (Because when you want to introduce a parameter object, it's usually because you have like 5 overloads of the method with different parameter combinations.) There should at least be an option to use initializer syntax instead.
