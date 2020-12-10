---
title: Exclude code regions from ReSharpers' format
date: 2020-09-17 22:55:34
tags:
  - ReSharper
  - code format
---

A dev-friend was unhappy about not being able to exclude certain code passages from R#'s cleanup/formatting feature.

Actually, you can:
```csharp
// @formatter:off 
// here goes the ugly code
// @formatter:on
```

Personally, I'm not a big fan of this. I know that occasionally R# fucks up the formatting. But I prefer to deal with this by trying to fiddle with the formatting settings. If that doesn't work, then it's either one of two scenarios:
a) a rare boundary case
b) a more common and thus really annoying case

If it's a), I just ignore it. If it's b), I create a support ticket with R#. 

My reasoning is that uniform and automated formatting is so valuable that I can live with rare instances of it not quite working as I'd wish. But if there's something that occurs more often, then I rather fix my settings or, worst case scenario, ask for a feature from R#, than going back to manually maintained code-formatting.


