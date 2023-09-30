---
date: 2023-09-30
categories:
  - Notes
---

# Python in neovim with pyenv

In contradiction to vim, neovim support for python3 is not added by compilation flag but by python3 package. This package is not installed by default and need to be installed manually.

Neovim instruction suggest installing it using `--user` flag. 
I do not like this suggestion as packages installed with `--user` flag are visible in all python environments.

During play with neovim I found that it scan all python environments and use first one that have `neovim` package installed.

So my preferred solution for now is to create separate python environment for neovim and install `neovim` package in it.

```shell
$ pyenv virtualenv 3.11.5 neovim
$ pyenv shell neovim
$ pip install pynvim
```

After that I can use `:checkhealth` command in neovim to check if everything is ok.

Please use python version already installed on your machine instead of `3.11.5` in above commands.