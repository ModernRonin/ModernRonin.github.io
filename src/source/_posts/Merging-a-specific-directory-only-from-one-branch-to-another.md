---
title: Merging a specific directory only from one branch to another
tags:
  - git
date: 2020-08-28 21:09:33
---


Sometimes, you find yourself in the situation of needing just one directory from one branch in another one. 

Depending on what you need and your circumstances, there are multiple ways to achieve that.

> You don't need the history of that directory - in this case the simplest way is:

```sh
git checkout <destinationBranch>
git checkout <sourceBranch> -- <directoryName>
```

> More often you will need the history. This is a bit more complicated:

```sh
git checkout <sourceBranch>
git subtree split -P <directoryName> -b tmpBranch
git checkout <destinationBranch>
git subtree merge add -P <directoryName> tmpBranch
```

> Sometimes, the last method will fail with a message about unrelated histories. In this case, there is the following method (assuming you are using PowerShell):
```powershell
git checkout <sourceBranch>
git log --pretty=format:"%h" --reverse -- <directoryName> > c:\tmp\commits.txtwith the oldest and writes them to commits.txt
git checkout <targetBranch>
foreach ($line in Get-Content C:\tmp\commits.txt) 
{
  git cherry-pick $line
  if (-not $?)
  {
    Read-Host "There are conflicts applying the commit. Fix them and come back here and press <Enter>"
  }
} 
```

This basically gets all commit hashes affecting the directory you are interested and saves them in a file. It saves them in reverse order (oldest first), so in the next step we can apply them via cherry-pick in historically correct order.

The main drawbacks of this method are that it can take a long while if you have a lot of relevant commits (for me, it's around 1 commit per second) and that if a commit affects both content in the relevant directory and outside of it, the stuff outside of it will also be merged into `destinationBranch`. This later issue depends a bit on your commit discipline, but if it happens it can be a huge problem. 
