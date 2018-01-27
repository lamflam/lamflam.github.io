---
title: replaying git commits
date: 2017-11-09 18:21:35
tags:
- TIL
- git
- rebase
---

#### TL;DR
```bash
# To replay the last N commits from current branch onto another branch
git rebase --onto other_branch HEAD~N
# Same as above, but does so on a NEW branch so your current branch is unaffected
git rebase --onto other_branch HEAD~N new_branch
# Same again, except replay the last N commits from some feature branch
git rebase --onto other_branch feature_branch~N new_branch
```
 
<br>
#### Replaying commits on another branch
Sometimes you may find yourself in a situation where you have done some work on a branch, but the starting point of
your branch is not what you wish. This is probably quite rare if you have a merge-heavy git workflow, but may be common
if your workflow involves a lot of rebasing onto a `master` branch. 

For example, say your workflow starts by creating a `feature` branch off `master`. You work on `feature` for a little
while, commit and send `feature` down the pipeline to be tested. Then you have some work that is not directly part of
`feature` but depends on the work done on that branch so you create a branch `feature+` off of `feature`. Now you have
three branches with the follow histories:

<img src="/img/replay-git-commits/initial.png" ></img>
<br />
<br />
<br />

While you're working someone merges a new commit to `master`.

<img src="/img/replay-git-commits/new_commit_master.png" ></img>
<br />
<br />
<br />

Now you need to pull these new changes into your work to make sure everything is working with the latest code, so you
rebase `feature` onto `master` and end up with the following:

<img src="/img/replay-git-commits/rebase_feature.png" ></img>
<br />
<br />
<br />

Depending on how complicated the changes were, using `git rebase feature` to rebase `feature+` onto `feature` may not
do what you want. Those branches no longer share the `F1` commit because `F1` was changed to `F1'` when `feature` was
rebased onto `master`. However, it is easy to use `git rebase` to replay just the commits you want on top of the latest `feature`.
In this case, while on the `feature+` branch you can run the following to replay the `F+` commit onto the current history
of `feature` and end up with the history that you want:
```bash
git rebase --onto feature HEAD~1
```

Now all three branches with the proper git histories.

<img src="/img/replay-git-commits/rebase_feature+.png"></img>
