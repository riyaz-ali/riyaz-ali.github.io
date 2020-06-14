---
title: Git Workflow
date: 2019-02-21
author: riyaz-ali
background: https://source.unsplash.com/842ofHC6MaI
description: |
    This article is meant to serve as a documentation around
    a common git workflow based on a _fork-clone-pull_ model
    and how to implement it in your project.
tags:
- SoftwareEngineering
- Git
keywords:
- SoftwareEngineering
- Git
- GitWorkflow
---

In this post I'm going to write about the _fork-clone-pull_ that we've been using for some time now at work.

To begin with, the Fork workflow (just workflow from now on) extend [Github's Workflow](https://guides.github.com/introduction/flow/index.html) and adds [forking](https://help.github.com/en/articles/fork-a-repo) to the mix..

## And why do that?

Several benefits.

- You don't need to give _everyone_ write access to the _central repository_
- It keeps the central repository "clean" (arguable..) by not polluting it with developers' feature branches
- Developers can form _subteams_ (by adding other dev's clone as a `remote`) (useful for collaborating, pair-programming etc.)
- _Enforce_ review guidelines by limiting direct write access
- Very useful for remote teams (personal experience..)

Convinced? nice! read ahead to know more about how to implement..

## Implementation

- [Organsiation](https://help.github.com/en/articles/about-organizations) admins <sup>[1]</sup> (or users with write access) start by creating the central repository (the `upstream`)
- Members than [fork the upstream](https://help.github.com/en/articles/fork-a-repo#fork-an-example-repository) under their _own account_ (the `origin` for that dev)
- The developers than start by [cloning](https://git-scm.com/docs/git-clone) their respective `origin` (each dev has complete access over their forks) and add the upstream remote  

```shell
git clone git@github.com:user/repo
git remote add upstream git@github.com:organisation/repo
```

- One can then follow the [git branching](https://nvie.com/posts/a-successful-git-branching-model/#supporting-branches) workflow that they are already familiar with.

```shell
git checkout -b feature/the-next-big-stuff
```

- Once they've started working on the feature, they can push it up to their fork.

```shell
git push -u origin HEAD
```

- Once they've pushed the code to `origin`, they can then open a [Pull Request](https://help.github.com/en/articles/about-pull-requests) to the `upstream` where the project owner / maintainer can review the code and merge it (or suggest any changes)

## Keeping the code in sync

When multiple developers are working on a project it's very likely that the `upstream` would move much faster than the devs' `origin`. In that case, the dev would need to periodically (ideally, they should do it daily) _sync_ their `origin` with the `upstream`.

To do that, devs need to execute following (on their local machine) -

```shell
# fetch upstream and all branches
> git fetch upstream
# checkout local or own develop
> git checkout develop
# get the changes from the upstream and merge to your local.
> git merge upstream/develop
# push the code to the develop branch of your fork
> git push origin develop
```

The devs should also execute ☝️ once they've their PR merged.

----------

And that's it! you've got a scalable git workflow model that would work well even for remote teams (we've used it quite extensively!)..  

And just to make it clear.. we are not the first ones who've used this or anything like that 😛.. this flow is used extensively in open-source world where generally there is a central `upstream` _canonical_ repository (where all issues are made and project is released) and each contributor has his/her own "fork" of that repository. The beauty of this flow is that it scales so well even for a highly distributed open source project!

## References

- [Understanding the GitHub Flow](https://guides.github.com/introduction/flow/index.html) by GitHub
- [A successful Git branching model](https://nvie.com/posts/a-successful-git-branching-model/) by Vincent Driessen @ [nvie](https://nvie.com/about/)
- A big shout to my friend [@vinitkme](https://twitter.com/vinitkme) for familiarising me with this!
