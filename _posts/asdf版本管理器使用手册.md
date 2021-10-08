---
title: asdf版本管理器使用手册
tags: 编程基础
categories: 编程基础
abbrlink: 7892430c
date: 2021-09-22 19:49:43
---

## Why use ASDF?
ASDF is a version manager for programming languages. It's somewhat like RVM is for Ruby or NVM is for Node but it also supports Erlang, Elixir, Haskel, Ocaml, PHP, Python, Rust and many other languages.

## Prereqs
This guide assumes you have homebrew and Xcode command line tools and nothing else. To see the setup of those from a fresh macOS Mojave install, see this short video.

## Clone the repo
The ASDF repo page on will have directions for cloning the newest repo github: https://github.com/asdf-vm/asdf

```
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.7.1
```

Next, add it to your shell:

```
echo -e '\n. $HOME/.asdf/asdf.sh' >> ~/.bashrc
echo -e '\n. $HOME/.asdf/completions/asdf.bash' >> ~/.bashrc
```

Restart your shell (or just open a new tab to work from) and run asdf to verify that it's installed.

## Install the plugin dependencies

```
brew install \
  coreutils automake autoconf openssl \
  libyaml readline libxslt libtool unixodbc \
  unzip curl gpg wxmac
```

## Basic commands
* asdf plugin-list-all: shows all the plugins (i.e., languages) available
* asdf plugin-add <language>: installs language
* asdf list-all <language>: shows all available versions of language
* asdf list <language>: shows all installed versions of language
* asdf install <language> <version>:
* asdf current: shows currently enabled languages
* asdf global <language> <version>: enables the chosen version of a language

## Installing Node
Node requires an extra step:

```
bash ~/.asdf/plugins/nodejs/bin/import-release-team-keyring
asdf plugin-add nodejs
asdf list-all nodejs
asdf install nodejs <version>
asdf global nodejs <version>
```

## Installing Erlang

```
asdf plugin-add erlang
asdf list-all erlang
```

According the readme on the ASDF Erlang plugin repo we need to pass a couple of flags to the install command:

```
export KERL_CONFIGURE_OPTIONS="--disable-debug --without-javac"
asdf install erlang <version>
asdf global erlang <version>
```

## Installing Elixir

```
asdf plugin-add elixir
asdf list-all elixir
asdf install elixir <version>
asdf global elixir <version>
```

## Installing Ruby

```
asdf plugin-add ruby
asdf list-all ruby
asdf install ruby <version>
asdf global ruby <version>
```

## Verify you have what you need

```
asdf current
```

## Installing the Phoenix Framework
If you're reading this on Alchemist Camp, you're likely using ASDF for Elixir and also want to set up Phoenix:

```
mix local.hex
mix archive.install hex phx_new 1.5.3 # or whichever version you prefer
```

## Done!
ASDF is a very handy tool for setting up dev machines and keeping the versions of whichever languages you may need. It's a big improvement to have one unified tool over several language specific ones.

ASDF also supports per-project configuration via `.tool-versions` files and a number of other things not covered in this setup guide.

## How to fully remove ASDF
If you ever want to fully remove ASDF from your system, all you need to do is delete the directory where you cloned it at the top of the tutorial and the .tool-versions (if any):

```
rm -rf ~/.asdf/ ~/.tool-versions
```

You'll probably want to get rid the lines with $HOME/.asdf/asdf.sh and $HOME/.asdf/completions/asdf.bash in your shell config, and that's all there is to it.
