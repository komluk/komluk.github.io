---
title: Working with multiple remotes repositories
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2022-11-29 15:00:00 +000
categories: [Git, Source control]
tags: [git]
render_with_liquid: false
---

This post will guide you how to work with `git` remotes and push to multiple repositories with a single command. If you don't know what [`git`](https://git-scm.com/) is, you should probably read about it before you continue.

## Prerequisites
Basic knowledge of the following `git` commands:
```bash
git init
git pull
git commit
git push
```
Have a write access to one or more remote `git` repositories

### Defining multiple remotes
The first step is to add remote repos to your project

```bash
git remote add REMOTE-ID REMOTE-URL
```
> Syntax to add a git remote

By convention, the original / primary remote repo is called origin. Here’s a real example:

```bash
git remote add origin git@github.com:komluk/repository.git
```
> Add remote 1: GitHub

```bash
git remote add upstream git@devops.com:komluk/repository.git
```
> Add remote 2: Azure DevOps

### Change remote url
If you want to change the URL associated to a remote that you’ve already added, you can do it with the following command:

```bash
git remote set-url upstream git@github.com:komluk/repository2.git
```

### List all remotes
To see a list of all remotes, simply use the following command:
```bash
git remote -v
origin	    git@github.com:komluk/repository.git (fetch)
origin	    git@github.com:komluk/repository.git (push)
upstream    git@devops.com:komluk/repository.git (fetch)
upstream    git@devops.com:komluk/repository.git (push)
```

### Push to multiple remotes
Now we have a primary remote repo and other remotes as well, it’s time to configure the push. The objective is to push to multiple Git remotes with a single git push command.

```bash
git remote add all git@github.com:komluk/repository.git
```
>Create a new remote called `all` with the URL of the primary repo.

```bash
git remote set-url --add --push all git@github.com:komluk/repository.git
```
>Re-register the remote as a push URL.

```bash
git remote set-url --add --push all git@devops.com:komluk/repository.git
```
>Add a push URL to a remote. This means that "git push" will also push to the second `git` repository.

Now, we can push to all remote repositories with a single command!
```bash
git push --all BRANCH
```
>Replace BRANCH with the name of the branch you want to push.

## Pull from multiple remotes

It is not possible to `git pull` from multiple repos. However, you can `git fetch` from multiple repos with the following command:
```bash
git fetch --all
git checkout BRANCH
```
>Checkout the branch you want to work with.
```bash
git merge remotes/upstream/main
```
>Merge from remotes to main branch
```bash
git reset --hard REMOTE-ID/BRANCH
```
>Reset the branch to match the state as on a specific remote.