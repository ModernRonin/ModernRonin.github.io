---
title: Multiple Visual Studio environments on one machine
date: 2020-08-12 16:19:30
tags:
  - Visual Studio
  - ReSharper
---

I just found out there is a way to have multiple VS environments (without actually having side-by-side installs).

From a prompt, run
```powershell
devenv /RootSuffix MyEnvironment
```

(This is assuming you got devenv on your PATH. Anyway, you probably will want to create a shortcut if you use this.)

Such a second environment can have its own settings, including extensions. So, for example, say you are working with a huge legacy codebase with code-practices that involve code-files with multiple 1000s of lines. As you have probably noticed painfully, in such code-bases ReSharper really gets very slow and sometimes even crashes VS. 

So now you can just create a new environment for VS that doesn't even have ReSharper installed, and you use this for working with said solution, while you use your regular environment for everything else.


