---
layout: post
title: "Improving workflow with git hooks"
comments: true
categories: [workflow]
---

## Workflow

The team I work in uses [git](https://www.git-scm.com/) day to day to manage [ticketsolves](http://www.ticketsolve.com/) codebase. In general the workflow is:

1. Create new feature branch.
2. Do work in branch.
3. Create a pull request.
4. Reviewer offers criticism and suggestions.
5. Update feature branch based on feedback.
6. Repeat steps 4-6 until everyone is happy.
7. Merge feature branch to master branch.

As well as functional updates to the code base often there are cosmetic changes and updates to syntax. For example [rubocop](http://batsov.com/rubocop/) is used to check the code conforms to style guidelines. But being a large legacy code base rubocop is only run against new and changed files, large number of files written before rubocop was introduced are not checked. Only when a change in a legacy file is made does rubocop start checking it. A small change made to an old file can result in many rubocop warnings and to fix these a lot of small style updates to a file that have nothing to do with the feature.

# Seperate Cosmetic from Functional Changes

To keep code review simple and the cosmetic changes seperate from the feature changes cosmetic changes should be added in seperate commits. Ideally all the cosmetic changes should be made in a single first commit to the branch. Consistently putting all the cosmetic changes in the first commit make it easier for the reviewer, they know where to expect all cosmetic changes.

Of course you don't know ahead of time what files you'll be touching or what cosmetic changes you'll make. This is where the `--fixup` commit flag is really useful. After the first cosmetic change, take a note of the commit sha `bcde234` and then for future cosmetic commits use `git commit --fixup bcde234`. Your history will look something like

```sh
$ git log --oneline --decorate
defg456 (HEAD, my-feature-branch) fixup! commit cosmetic
cdef345 commit 3
bcde234 commit cosmetic
abcd123 commit 1
wxyz789 (master)
```

Now your ready to make a pull request, before you do so tidy up your git history by squashing the cosmetic commits together.


```sh
$ git rebase --interactive --autosquash abcd123^
```

Git will open your editor with the `fixup` commit in the correct place and marked to fixup.

```sh
pick abcd123 commit 1
pick bcde234 commit cosmetic
fixup defg456  fixup! commit cosmetic
pick cdef345 commit 3
```

# Automate

Git [hooks](https://www.git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) are a way to trigger custom scripts when certain actions occur. For instance [`post-checkout`](https://www.kernel.org/pub/software/scm/git/docs/githooks.html#_post_checkout) is run after `git checkout` is performed. This hook can be used to assist in my workflow and remind me to put my cosmetic changes in a single seperate commit.

`.git/hooks/post-checkout`

```sh
#!/bin/bash
if test "$3" == "0"; then exit; fi

BRANCH_NAME=$(git symbolic-ref --short -q HEAD)
NUM_CHECKOUTS=$(git reflog | grep -o ${BRANCH_NAME} | wc -l)
MESSAGE="To commit cosmetic changes:\n git commit --fixup £1\nBefore pull request run:\n git rebase --interactive --keep-empty --autosquash £1^\nOr if no cosmetic chagnes made:\n git rebase --interactive £1"

#if the refs of the previous and new heads are the same
#and the number of checkouts equals one a new branch has been created
if [ "$1" == "$2"  ] && [ ${NUM_CHECKOUTS} -eq 1 ]; then
  git commit --allow-empty -m "Non functional updates"
  COSMETIC_COMMIT=$(git rev-parse --short --verify HEAD)
  echo -e ${MESSAGE//£1/$COSMETIC_COMMIT}
fi
```

Running `git checkout -b MRLOOP_new_feature_branch` a new empty commit is created and instructions are printed to the console telling you how to add new cosmetic changes commits and how to easily rebase them before a pull request is made.

```sh
$ git checkout -b MRLOOP_new_feature_branch
Switched to a new branch 'MRLOOP_new_feature_branch'
[MRLOOP_new_feature_branch f3b7a19] Non functional updates
To commit cosmetic changes:
 git commit --fixup f3b7a19
Before pull request run:
 git rebase --keep-empty --interactive --autosquash f3b7a19^
Or if no cosmetic changes made:
 git rebase --interactive f3b7a19^
```

If no cosmetic changes are made then `git rebase --interactive` without the `--keep-empty` flag and the empty cosmetic change commit is not picked.

```sh
#pick f3b7a19 Non functional updates
pick ob1cd34 first functional commit
pick gh22f45 second functional commit
pick vef44hi third functional commit
```
