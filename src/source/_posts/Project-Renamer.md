---
title: Project Renamer
date: 2020-09-04 22:04:13
tags:
  - renameproject
  - commandline tools
  - Visual Studio
---


Okay, this is going to be a bit of a plug, but...

Over the years, it has always annoyed me that Visual Studio doesn't properly support renaming projects. Properly as in: also renaming the directory, the project file, updating the references and keeping the git history intact. 

Recently, I found myself once again googling to find whether there is already some tool doing this properly. And it turns out, no, this is still unsolved and at the same time obviously in high demand. See for example this [StackOverflow post](https://stackoverflow.com/questions/211241/how-can-i-rename-a-project-folder-from-within-visual-studio) or this [Gist](https://gist.github.com/n3dst4/b932117f3453cc6c56be).

So I decided to create a little tool for this. 

If you find yourself in the same situation, just install it with

```sh
dotnet tool install -g ModernRonin.ProjectRenamer
```

and head over to the corresponding [GitHub repo](https://github.com/ModernRonin/ProjectRenamer) for documentation.