---
title: "Syncing a fork with upstream"
date: 2021-02-03T10:16:19+07:00
toc: true
tldr: Configure a remote for a fork by adding the upstream `git remote add upstream https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git` then make sure the remote has been added by listing remotes `git remote -v`. Fetch the branches and commits from upstream `git fetch upstream`, checkout to your forks default branch and merge the changes 'git checkout main' and 'git merge upstream/main'.
tags: ["git","fork","workflow"]
---

## Getting started

By now you're probably familiar with `git`. Git is a popular distributed version control system with which people can collaborate on pretty much anything that you can/want to version control and collaborate on. This can be source code, documentation, automation scripts, anything really. To work on a project with others, one would generally create a fork of the main repository where the code, it's branches and issues are being stored at. This short article explains what to do when the upstream project pulls too far ahead that it leaves your fork out-of-date creating potential issues during a pull request (PR) which would have to be resolved by the person who the PR is assigned to.  

What you need:
1. A terminal emulator
1. A fork of a project
1. The URL of the upstream project

## Steps to sync with upstream
To sync your fork with the upstream follow these steps:
1. List the currently configured remotes for your project.
```
$ git remote -v
origin  git@github.com:USERNAME/FORK.git (fetch)
origin  git@github.com:USERNAME/FORK.git (push)
``` 
2. Add a new remote *upstream* repositroy that you will sync with your fork.
```
$ git remote add upstream https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git
```
3. Make sure that the remote has been added.
```
$ git remote -v
> origin    https://github.com/USERNAME/FORK.git (fetch)
> origin    https://github.com/USERNAME/FORK.git (push)
> upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (fetch)
> upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (push)
 
``` 
4. Fetch the upstream branches and commits.
```
$ git fetch upstream
> remote: Counting objects: 75, done.
> remote: Compressing objects: 100% (53/53), done.
> remote: Total 62 (delta 27), reused 44 (delta 9)
> Unpacking objects: 100% (62/62), done.
> From https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY
>  * [new branch]      main     -> upstream/main
```
5. Check out you fork's default branch.
```
$ git checkout main 
Switched to branch 'main'
Your branch is up to date with 'origin/main'.
```
6. Merge the upstream changes to your local fork's main repository.
```
$ git merge upstream/main
```

## Conclusion
With these few steps you can keep your fork up-to-date with the upstream project hopefully causing less overhead to the upstream maintainers.
