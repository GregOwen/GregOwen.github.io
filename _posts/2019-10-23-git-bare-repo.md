---
title: "What are bare Git repos and what are they used for?"
date: 2019-10-23T14:46:00-07:00
classes: wide
categories:
  - blog
tags:
  - dotfiles
  - git
  - tools
---

After having to manually reconfigure all of my dotfiles for the 3rd or 4th time, I decided it was finally time to put them into version control. While I was poking around the internet to see how other people did this, I came across [this post](https://news.ycombinator.com/item?id=11070797) on Hacker News that described how to use Git's "bare repository" feature to track my dotfiles without turning my home folder into a Git repo.

This sounded great, but I had never heard of a bare repository before and so I wanted to understand more before I committed all of my dotfiles to this thing. After sifting through piles of blog posts and StackOverflow answers, I remained unenlightened. I hope this post will be helpful to anybody else who wants to know what the heck a bare repository is and when you would want to use one.

# What is a bare repository?

I assume that you've worked with a Git repo before. You've probably run `git init` to create a new repository on your local machine or run `git clone` to download a repo from GitHub/GitLab/BitBucket. You've probably even worked on different branches, made pull requests, and gone back to old commits to figure out why the heck you added some code that ended up causing a bug several months later.

That repo you've been using is a **non-bare repository**. It contains two pieces of state:

1. A **snapshot** of all of the files in the repository (in Git jargon, this is the "working tree")
1. A **history** of all changes made to all the files that have ever been in the repository (there doesn't seem to be a concise piece of Git jargon that encompasses all of this)

The **snapshot** is what you probably think of as your project: your code files, build files, helper scripts, and anything else you version with Git. I call this a snapshot because it's what your files happen to look like at one point in time. If you check out a different commit, the files change: you're looking at a snapshot from a different point in time.

The **history** is what allows you to check out that other commit and get a complete snapshot of what the files in your repository looked like when that commit was added. It's internal to Git and you've probably never interacted with it directly.

In a non-bare repository, the history is stored in the `.git` folder at the top level of your repository (I encourage you to poke around in it by running `tree .git`[^repository-layout] - go ahead, I'll wait). The most important piece to understand here is that the history doesn't just store metadata (e.g. "User U added this many lines to File F at Time T as part of Commit C"), it also stores data (e.g. "User U added *these exact lines* to File F").

The key idea of a bare repository is that you don't actually need to have the snapshot. Git keeps the snapshot around because it's convenient for humans and other non-Git processes that want to interact with your code[^non-git-processes], but the snapshot is just duplicating state that's already in the history. This is why you can delete all of the files in your snapshot and then restore them with `git checkout -- .`: the data is still in the history even if you delete the snapshot.

A **bare repository** is a Git repository that does not have a snapshot. It just stores the history. It also happens to store the history in a slightly different way[^different-storage], but that's not nearly as important.

A bare repository will still store your files (the history has enough data to reconstruct the state of your files at any commit). You can also create a non-bare repository from a bare repository: if you `git clone` a bare repository, Git will automatically create a snapshot for you in the new repository (if you want a bare repository, use `git clone --bare`).

# So why would we use a bare Git repository?

Almost every explanation I found of bare repositories mentioned that they're used for centralized storage of a repository that you want to share between multiple users[^shared-repo]. Basically, if you wanted to write your own GitHub/GitLab/BitBucket, your centralized service would store each repo as a bare repository. But why? How does not having a snapshot connect to sharing?

The answer is that there's no need to have a snapshot if the only service that's interacting with your repo is Git. Basically, the snapshot is a convenience for humans and non-Git tools, but Git only interacts with the history. Your centralized Git hosting service will only interact with the repos through Git commands, so why bother materializing snapshots all the time? The snapshots only take up extra space for no gain[^snapshot-makes-no-sense].

# Using a bare Git repository to track dotfiles

Bringing us back full-circle, we can now look at why you'd want a bare Git repo for your dotfiles.

We already covered the fact that Git repos don't need to store a snapshot of the files tracked by the repo. More generally, Git repos don't need to store that snapshot *in the same folder as the history*. You can store your history in one folder and your snapshot in a different folder.

This is why it's nice to use a bare repo to track dotfiles (or any set of files that already exist in some location that you don't want to turn into a Git repo): you can leave the dotfiles where they are (so that they'll still get picked up by the different tools that use them) while still getting the benefits of version control. You don't need to manually link dotfiles from your Git repo to their correct locations, and you don't need to turn your home directory into a Git repo[^no-home-git-repo].

To set yourself up with a bare git repo for your dotfiles, do the following[^tested-on-macos]:

1. Create a new bare Git repo to store the history for your dotfiles

        git init --bare $HOME/dotfiles

1. Tell Git that it should use your home directory as the working tree (snapshot) for this bare Git repo (it's often more convenient to create an aliased command rather than specifying this option manually on every Git command)

        alias dotgit='git --git-dir=$HOME/dotfiles/ --work-tree=$HOME'

1. Tell Git that this repo should not display all untracked files (otherwise `git status` will include every file in your home directory, which will make it unusable)

        dotgit config status.showUntrackedFiles no

1. Set up a remote repo to sync your dotfiles to (I use GitHub)

        dotgit remote add origin git@github.com:GregOwen/dotfiles.git

1. Whenever you want to add a new dotfile to your Git repo, use your aliased Git command with your special options set

    ```
    dotgit add ~/.gitconfig ~/.gitignore_global
    dotgit commit -m "Git dotfiles"
    dotgit push origin master
    ```

To download your dotfiles onto a new machine, do the following:

1. Clone your repo into a temporary location as a non-bare repository (since you're trying to move the actual dotfiles onto the new machine, you need to have a snapshot of the actual dotfiles, not just the history)

        git clone --separate-git-dir=$HOME/dotfiles git@github.com:GregOwen/dotfiles.git dotfiles-tmp

1. Copy the snapshot from your temporary directory to the correct locations on your new machine

        rsync --recursive --verbose --exclude '.git' dotfiles-tmp/ $HOME/

1. Remove the temporary directory

        rm -rf dotfiles-tmp

1. To allow your new machine to track changes to the dotfiles, follow the setup steps listed above (at this point, your new machine has the dotfiles in the correct locations but isn't tracking them, which is exactly the same state that your original machine was in before we did this setup)

For a handy cheat sheet, see GitHub user Siilwyn's writeup [here](https://github.com/Siilwyn/my-dotfiles/tree/master/.my-dotfiles).



[^repository-layout]: You can also get a high-level (if somewhat jargony) explanation of what all the subfolders of `.git` do by running `git help repository-layout`.

[^non-git-processes]: For example, you could theoretically edit your files by changing Git's object store directly. Your build process could theoretically read Git's object stores to get the current state of all code files and use that to build an executable. That would be super painful, though, and it would require any process that operates on a Git repository to be able to understand Git's storage model. As a corollary, it would also make Git's storage model an external-facing interface, which would make it much harder for the developers of Git to change the storage model without breaking everything. You can see why Git provides a snapshot by default.

[^different-storage]: In a non-bare repository, all of the history is stored in subdirectories of the `.git` directory (e.g. for a project called `MyProject`, this data would be in `MyProject/.git/objects`, `MyProject/.git/refs`, etc.). In a bare repository, the history is stored in multiple top-level directories at the project root (e.g. `MyProject/objects`, `MyProject/refs`, etc.). The layout is otherwise the same.

[^shared-repo]: This use case is actually suggested by `git help repository-layout` itself:

    > A Git repository comes in two different flavours:
    >
    > - a `.git` directory at the root of the working tree;
    >
    > - a `<project>.git` directory that is a bare repository (i.e. without its own working tree), that is typically used for exchanging histories with others by pushing into it and fetching from it.

[^snapshot-makes-no-sense]: Also, the concept of a snapshot doesn't really make sense when multiple different people are interacting with the repo. If I'm working on one branch of the repo and you're working on a different branch, which branch should the snapshot be of?

[^no-home-git-repo]: There's no reason that this would be impossible, but it would be annoying for a few reasons. It would mean that every subdirectory of your home directory would be part of the home git repo, so accidental git commands would be valid rather than triggering the helpful "you're not in a Git repo" error. It would also mean putting a `.git` directory in your home directory, which seems to me like unnecessary clutter.

[^tested-on-macos]: I tested all these commands on MacOS, but they should work for any Unix-like operating system