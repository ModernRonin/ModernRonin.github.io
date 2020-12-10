---
title: renameproject v2.0
date: 2020-10-20 23:02:01
tags:
  - renameproject
  - commandline tools
  - Visual Studio
---

There's an update out for `renameproject`. This is a bit of a bigger change in how you control its behavior. See [here](https://github.com/ModernRonin/ProjectRenamer#release-history) for details, or just update it with

```sh
dotnet tool update --global ModernRonin.ProjectRenamer
```
and call it without arguments from your solution directory to see it's shiny new help.

This makes use now of [FluentArgumentParser](https://github.com/ModernRonin/FluentArgumentParser).



