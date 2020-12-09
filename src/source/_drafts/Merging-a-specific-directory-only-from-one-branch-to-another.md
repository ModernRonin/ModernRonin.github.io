---
title: Merging a specific directory only from one branch to another
date: 2020-08-28 13:09:33
tags:
 - git
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
git subtree merge 
```
