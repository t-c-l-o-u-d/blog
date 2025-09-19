---
layout: post
title: "Global .gitignore"
date: 2025-03-08
tags: git macOS
---

A global `.gitignore` file allows you to specify patterns for files and directories that should be ignored by Git across all repositories on your system.

### macOS and .DS_Store
macOS creates a `.DS_Store` file inside any directory you view with Finder. This has been something that has annoyed me about using my MacBook for casual editing. After reading `man git-config`, I came across a method to ignore this across all repositories instead of modifying each one with a `.gitignore`.

> core.excludesFile -  
Specifies the pathname to the file that contains patterns to describe paths that are not meant to be tracked, in addition to .gitignore (per-directory) and .git/info/exclude. Defaults to $XDG_CONFIG_HOME/git/ignore. If $XDG_CONFIG_HOME is either not set or empty, $HOME/.config/git/ignore is used instead. See gitignore(5).

Success! This is exactly what I needed. Without further delay, lets see this in action.

### Here is a step-by-step guide to create a global .gitignore file:
1. Create the `.config` directory and the accompanying `ignore` file. Since this is macOS, you will not have any of the `XDG` variables, so we substitute them for their fallback.

    ```bash
    $ mkdir --parents ~/.config/git/
    $ touch ~/.config/git/ignore
    ```

2. Add the patterns for the files you want to ignore, in my case the dreaded `.DS_Store` files. You can also take this opportunity to include other things you wish to ignore globally, such as `.swp` files come to mind if you are a heavy `Vim` user.

    Contents of `~/.config/git/ignore`:
    ```bash
    # Ignore macOS system files
    .DS_Store
    ```

3. Verify things are working by checking for the presence of a `.DS_Store` in a git directory.

    ```bash
    tcloud@MacBookPro devcontainer-base-images % ls -1A
    .DS_Store
    .git
    .github
    .gitmodules
    .submodules
    .vscode
    COPYING
    Containerfile.ArchLinux
    Containerfile.Fedora
    Containerfile.UbuntuLTS
    Containerfile.template
    README.md
    devcontainer.json.template

    tcloud@MacBookPro devcontainer-base-images % git status
    On branch main
    Your branch is up to date with 'origin/main'.

    nothing to commit, working tree clean
    ```
    No `.gitignore` for this repository and no pending changes!

### Wrapping Up
This wasn't a particulary exciting post, but it was something that has bothered me for a while. I was just recently bothered enough by it to investigate and thought I should document it here for others. A special shout out to a colleague of mine for making me pursue this. Without their bug report on my devcontainer project, I may not have investigated this issue.
