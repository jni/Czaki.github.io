---
date: 2023-09-29
categories:
  - Notes
---

# Chezmoi for dotfiles management

When looking for a dotfiles manager I was looking for something that will be:

1. Easy to use
2. Allow to handle machine specific configuration
3. Keep dotfiles synchronization with only few commands

After read few articles I decided to try chezmoi. 

## Features 

Most important features of chezmoi for me are:

- Could be easy installed on Ubuntu and macOS
- Allow using templates for configuration files
- Allow specifying external dependencies like:
    - oh-my-zsh
    - Vundle
    - selected zsh plugins that is not installed by oh-my-zsh
    - language dictionary for vim
- configuration in text files
- use git for synchronization
- preview changes before applying them


## Configuration

- installation [https://www.chezmoi.io/install/](https://www.chezmoi.io/install/)
- external dependencies [https://www.chezmoi.io/user-guide/include-files-from-elsewhere/](https://www.chezmoi.io/user-guide/include-files-from-elsewhere/), may need to use [.chezmoiignore](https://www.chezmoi.io/reference/special-files-and-directories/chezmoiignore/) to exclude cache files


