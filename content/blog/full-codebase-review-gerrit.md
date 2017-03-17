+++
date = "2017-03-16T19:56:53Z"
title = "Full codebase review with Gerrit"
draft = true

+++

* Create a new project in gerrit
* Push our original master branch.
```
git push ssh://tcolgate@src.curlpipesh.it:29418/golorp empty:refs/heads/empty
```
* We'll need a Change-Id in our future commit messages
```
scp -p -P 29418 tcolgate@src.curlpipesh.it:hooks/commit-msg .git/hooks/
```
* Create a new empty branch for the review
```
git checkout --orphan review
```
* Create a new empty branch for the review
```
git rm -r --cached *
git clean -fxd
git commit --allow-empty -m "Empty commit"
```
* Create an empty branch we will use for the diff.
```
git checkout -b empty
```
* Push the empty branch to gerrit as a new ref
```
git push ssh://tcolgate@src.curlpipesh.it:29418/golorp empty:refs/heads/empty
```
* Merge master into review
```
git checkout review
git merge --allow-unrelated-histories  master
git reset $(git merge-base  HEAD empty)
git add .
git commit -m "Review Everything"
```
* Push our new branch for review
```
git push ssh://tcolgate@src.curlpipesh.it:29418/golorp review:refs/for/empty
```

We're now ready to review! Log into the gerrit UI and there's a change request,
with out entire codebase waiting for us.

