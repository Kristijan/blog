---
title: "TMUX Flash Copy"
date: "2026-01-12"
categories: 
  - "tmux"
tags: 
  - "python"
  - "tmux"
---

There is a popular neovim plugin called [flash.nvim](https://github.com/folke/flash.nvim){:target="_blank"}, which amongst other things, allows you to quickly jump to a word in your code. It's a great tool for developers who want to quickly navigate their codebase.

I was looking for something similar, but for [tmux](https://github.com/tmux/tmux){:target="_blank"}. I wanted to be able to search visible words in the current tmux pane, then copy that word to the system clipboard by pressing the associated label key.

I built [https://github.com/Kristijan/flash-copy.tmux](https://github.com/Kristijan/flash-copy.tmux){:target="_blank"}, with the aim to bring that functionality to tmux.

![Demonstration of tmux-flash-copy in action](assets/images/tmux-flash-copy-demo.gif)

In the demonstration above, you can see how it works.
1. I launch the TMUX Flash Copy plugin using the default bind key `<tmux prefix> Shift + f`
2. The word `option_override` is what I want to copy, so I start searching for it. You can see as I search, words are highlighted in the pane and labels are assigned with each match.
3. I press the label key `j` associated with the word `option_override`, and it's copied to the system clipboard.

> You can automatically paste the copied text by holding the semicolon `;` modifier key while selecting the label.
{: .prompt-tip }

I tried to implement as many of the visual features from flash.nvim as possible, and added a few of my own which made sense:
- Dynamic Search: Type to filter words in real-time as you search.
- Overlay Labels: Single key selection with labels overlayed on matches in the pane.
- Dimmed Display: Non-matching content is dimmed for visual focus.
- Clipboard Copy: Selected text is immediately copied to the system clipboard.
- Auto-paste Modifier: Use semicolon key as a modifier to automatically paste selected text.
- Configurable Word Boundaries: Honours tmux's `word-separators` by default, with override support.

There are many customisable options available, including:
- Customising the prompt position, indicator and colour.
- Changing the colours used for highlighted matches and labels.
- Use either case sensitive or case insensitive search.

You can find a lot more detail and configuration options in the [README](https://github.com/Kristijan/flash-copy.tmux/blob/master/README.md). I welcome any feedback or contributions.
