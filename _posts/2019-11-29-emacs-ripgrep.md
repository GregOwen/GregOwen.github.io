---
title: "Blazing-fast jump-to-grep in Emacs using ripgrep"
date: 2019-11-29T07:30:00-08:00
editors:
  - Bianca Homberg
classes: wide
categories:
  - programming
tags:
  - emacs
  - git
  - grep
  - ripgrep
  - tools
---

I've recently started using [ripgrep](https://github.com/BurntSushi/ripgrep), a modern `grep` replacement written in Rust, and it's fantastic. It's just [so fast](https://blog.burntsushi.net/ripgrep/). It may not sound like much, but there's a huge difference between waiting a few seconds for search results and having them instantly: it means that searching doesn't break my flow.

After trying ripgrep from the command line, what I really wanted was to call it from within Emacs, find results across my entire Git repo, and jump to the matches it found ("jump-to-grep") without having to switch back to the terminal. Here's how to make that happen.

# Instructions

1. Install ripgrep. Follow the instructions on the [GitHub repo](https://github.com/BurntSushi/ripgrep#installation)

1. Make sure that `rg` (the ripgrep executable) is on your path

        which rg

    This should return a path to the rg executable. If it doesn't, figure out where your installer put the executable and add that folder to your `$PATH`.

1. Tell Emacs to use ripgrep instead of its default grep command. Add the following to your `~/.emacs` file, your `~/.emacs.d/init.el` file, or wherever you put your custom Emacs configuration

    ```
    (grep-apply-setting
      'grep-find-command
      '("rg -n -H --no-heading -e '' $(git rev-parse --show-toplevel || pwd)" . 27)
    )
    ```

    There's a lot going on here, so let's unpack it. At a high level, we're using the `grep-apply-setting` function to tell Emacs what command should be run when we run `M-x grep-find`. The value that we're passing is this complicated object `'("rg -n -H --no-heading -e '' $(git rev-parse --show-toplevel || pwd)" . 27)`. Let's look at what that means.

    To start, we're calling `rg` (this was our main goal: to call ripgrep).

    `-n` tells ripgrep to display the line number for each match (the line in the file on which the match appeared). `-H` tells ripgrep to display the file name for each match. `--no-heading` tells ripgrep to display the file name on each line, rather than grouping matches by file and only displaying the filename at the top of each group. This is important because Emacs will only let us jump to a match if we have the complete file path and line number all together.

    `-e` tells ripgrep that the next argument should be regular expression to search for. In this command, we start with an empty regular expression (`''`), which we will fill in whenever we call `M-x grep-find`.

    The final argument we supply to `rg` is the directory to search. `rg` will default to searching the current directory, but I want it to check the entire current Git repo (I want to see results from my whole project, not just the directory of the file I happen to have open). If we're in a Git repo we can use `git rev-parse --show-toplevel` to get the repo's root directory, but this command will return an error if we're not in a Git repo. We could just leave it there, but in the rare case that I'm searching outside of a Git repo I don't want this command to return an error.

    Bash (like most shells) allows us to recover from errors using the `||` operator. `cmd1 || cmd2` will run `cmd1`, then run `cmd2` if `cmd1` returned an error. In this case, our second command is `pwd`, which will print the directory we're currently in (`pwd` is short for Print Working Directory). So `$(git rev-parse --show-toplevel || pwd)` will be the root directory of the current Git repo if we're in a Git repo, or the current directory if we're not.

    So why do we have to include this weird `. 27` rather than just passing the command as a string? That part just tells Emacs where we want our cursor to start when we call `M-x grep-find`. In this case, we're starting the cursor at the 27th character of the string - right between the `''`, which is where we'll type our search expression.

1. Add a nicer keybinding than `M-x grep-find`. I expect to use grep a lot, and I'd rather not have to type `M-x grep-find` every time. I'd prefer something shorter, like `C-x C-g`. Add the following to your Emacs configuration file

        (global-set-key (kbd "C-x C-g") 'grep-find)

1. Try it out. Save and exit Emacs, then open up a file in some subdirectory of one of your Git repositories. Press `C-x C-g` and type a search term. You should see results from across your entire Git repo, not just the subdirectory you're currently in. If you move the cursor over one of the links and hit enter, Emacs should open that file and take you directly to that line.

# Learning to fish

Every time I spelunk through my Emacs config to get something working, I try to learn a little bit more about how Emacs works and how I can get better at modifying it to suit my needs. Here are the steps I took to figure out what I needed to do to get jump-to-grep working.

My first step was to understand how Emacs's grep integration works. I knew a little about the different [grep commands](https://www.gnu.org/software/emacs/manual/html_node/emacs/Grep-Searching.html) that Emacs exposes, but I didn't know much about how I could customize them. So from an Emacs buffer, I ran `M-x customize-group RET grep`. This opened a help buffer that explained that the default Emacs grep library allows you to change the command that appears when you run `M-x grep-find` by setting the variable `grep-find-command`. The help buffer also explained that I could set this value using `grep-apply-setting`.

Finding the correct arguments to `rg` was as easy as reading `rg --help`.

*Thanks to Bianca Homberg for giving me feedback on this post*