## Introduction

I have been revamping my terminal workflow recently, for no other reason than I wanted to. Having spent years on VS Code, I decided to commence editor hopping to find the right terminal workflow. I'm going to take the the time here to document what I've figured out and what I believe I have settled on for the medium/long term.

__Note__: Both of my development machines are Macbooks, so these will be MacOS-focused. However, I follow unix standard closely and so most of these practices should port nicely.

## Required

Let me start by listing out which tools I use in the terminal, from the big picture to the small picture. You will notice a theme in these tools: Low configuration and low magic where useful features work out of the box. These are listed roughly in the order in which they would be installed.

* __Homebrew__: My MacOS package manager.
* __Fish__: A low-configuration, feature-rish shell.
* __Alacrity__: My terminal emulator of choice. I used iTerm2, Kitty and Wezterm, which all had their strengths. iTerm2 is rock-solid stable. Kitty is chock full of features. Wezterm is has a happy middle spot, but it is under hot development has _minor_ perforamnce issues compared to the other two. However, I settled on Alacritty because I wanted something easy to configure. It doens't support panes, but we will solve that later.
* __Helix__: This is my editor of choice for the moment. After spending much time working on Neovim configuration, I decided once again I wanted something that is less difficult to configure. Helix is _amazing_. It works like a charm out of box, the documentation is great, it has built-in support for language serves, and is _easy as hell to configure_. Helix is magical. It is missing a few features, most primarily a file tree. Another problem to solve later.
* __chezmoi__: A dotfiles manager.
* __lazygit__: The king of Git terminal UIs (much better than the CLI, I swear).
* __zellij__: Terminal multiplexing (solves Alacritty's lack of splitting panes).

## Nice To Haves

The above are the bare minimum required to have a useful setup in the terminal for myself. Here are some more tools which are optional but nice:

* __starship__: Fancy terminal prompts.
* __bat__: A `cat` replacement. Alias to cat!
* __duf__: A `df` replacement. Shows free space.
* __dust__: Shows a pretty tree of the folders that are using the most space on your machine.
* __fzf__: A fuzzy finder, useful for many things.
* __glow__: A pretty terminal markdown renderer.
* __httpie__: An ergonomic `curl` replacement.
* __jq__: A JSON query machine.
* __lsd__: Fancy `ls` replacement. Alias to ls!
* __ripgrep__: A better (best) `grep`.
* __sd__: A simple, no-frills `sed` substitute.
* __tealdeer__: A `man` substitutes which shows your simple, useful examples rather than full manuals.
* __xplr__: A rad terminal file explorer, nice and customizable with low effort.
