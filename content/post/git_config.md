---
title: "Handy Git Config setttings"
date: 2025-06-03T21:38:57+10:00
draft: false
---

This is my self reference of handy `git config` settings to apply for whatever computer I'm setting up.


## Make git push just push a corresponding branch for new branches, instead of giving me something to copy past to do that

[`git config --global push.autoSetupRemote true`](https://git-scm.com/docs/git-config#Documentation/git-config.txt-pushautoSetupRemote)

The one place I don't use this is for open source repos where there's a fork workflow and I'm a maintainer on GitHub.
Here I'm often using the github cli `gh pr checkout #pr_id` and I'm paranoid about accidentally pushing unintended things
to other peoples forks (this is probably fine, but I don't know for sure)


## Make it so that git will remember resolved merge conflicts I've done as part of a rebase or merge, even if I abort that part way through

[`git config --global rerere.enabled true`](https://git-scm.com/book/en/v2/Git-Tools-Rerere)

This puts resolved hunks somewhere in the .git folder, so you can get git to "unremember" if it's remembering something
you resolved incorrectly

## Submodules stuff

### Make git pull pull submodules by default

[`git config --global submodule.recurse true`](https://git-scm.com/book/en/v2/Git-Tools-Submodules#_pulling_upstream_changes_from_the_project_remote)

Aside to mention that if you've got local changes, you can force the merging of them with `git submodule update --remote --merge`


### Make sure you can\'t push a commit of a parent repo if it references unpushed commits of submodules

[`git config --global push.recurseSubmodules on-demand`](https://git-scm.com/book/en/v2/Git-Tools-Submodules#_publishing_submodules)


## Other things in my gitconfig

Things I need to think more thoroughly about but don't have time for right now

```
[push]
#   these hopefully work for geopandas / decentralised workflows too
    # default = "simple" # default since git 2
    default = "current" # better for fork workflows?
    autoSetupRemote = "true"

```