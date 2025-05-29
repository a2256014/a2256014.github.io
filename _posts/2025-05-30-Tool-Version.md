---
layout: post
title: titletest
subtitle: subtitletest
author: g3rm
categories: Setting
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - Java
  - Python
  - jenv
sidebar:
---

## Intro
자바나 파이썬 등 버전을 바꿔서 사용할 가능성이 높은 것들 여러 버전으로 관리하기 위해 작성

## Java
```shell
brew install cask

# jdk 21
brew install --cask temurin@21

java --version

# jenv
brew install jenv
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(jenv init -)"' >> ~/.zshrc
jenv enable-plugin export

exec $SHELL -l
jenv add /Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home
jenv global 21.0

jenv versions
```

## Python
```shell
brew install pyenv

# idea ~/.zshrc
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"
eval "$(pyenv init -)"
```











