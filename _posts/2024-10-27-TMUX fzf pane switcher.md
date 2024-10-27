---
title: "TMUX fzf pane switcher"
date: "2024-10-27"
categories: 
  - "tmux"
tags: 
  - "fzf"
  - "scripting"
  - "shell-scripting"
  - "tmux"
---

If you currently use [screen](https://github.com/alexander-naumov/gnu-screen){:target="_blank"} as your terminal window manager, you should take a look at [tmux](https://github.com/tmux/tmux){:target="_blank"}. This isn't a screen vs tmux post, but I made the switch a few years back, and I've become a lot more productive as a result of it.

tmux has a concept of sessions, windows, and panes. A session is a collection of one or more windows, and a window can contain one or more panes. At work, I use sessions a lot. I'll have a session that contains all my VIOS, another for all my NIM's, and another for all my test servers. To switch from one session to another, you press `prefix + s` to bring up the current list of sessions, arrow down to the session, and hit enter. You can expand the session list to show all the windows, and from there further select the exact window from the session you want to go to.

This flow I find time consuming. I'd much rather bring up the list of sessions, and search to narrow down where I want to switch to. So piggy backing on another tool that I use, I wrote a small little tmux plugin that brings up a fzf window with my sessions, windows, and panes, and lets me quickly get to the pane I want.

[tmux-fzf-pane-switch](https://github.com/Kristijan/tmux-fzf-pane-switch){:target="_blank"}

Using the plugin with the default mapping of `prefix + s`, I now get a [fzf](https://github.com/junegunn/fzf){:target="_blank"} window where I can narrow my search by filtering on the `#{window_name}`, `#{pane_title}`, or `#{pane_current_command}`. I've got a demo video below that shows it in action.

{% asciicast lRfrNLEL5WhqAgNsMnNw4zxY7 %}

If you use tmux, hopefully you'll see the benefit in using it.
