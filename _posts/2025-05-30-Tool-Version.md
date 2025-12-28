---
layout: post
title: Python & Java 버전 관리
subtitle: Python & Java 버전 관리
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
  - Gradle
  - Mac
sidebar:
---
## Intro
자바나 파이썬 등 버전을 바꿔서 사용할 가능성이 높은 것들 여러 버전으로 관리하기 위해 작성

## Java & Gradle
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

# Gradle
brew install gradle

```

## Python
```shell
brew install pyenv

# idea ~/.zshrc
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"

pyenv --version

# 설치 가능한 Python 버전 확인
pyenv install --list

# 특정 버전 python 설치
pyenv install 3.12.0

# 특정 버전 Python 삭제
pyenv uninstall 3.12.0

# 설치된 Python list 확인하기
pyenv versions

# 원하는 Python 버전을 기본으로 설정하기
pyenv global 3.12.0

# 특정 위치에서 원하는 Python 버전 사용하기
pyenv local 3.12.0

# 가상환경 추가
pyenv virtualenv 3.12.0 py3.12.0
# 가상환경 실행
pyenv activate py3_10_4
# 가상환경 나오기
source deactivate
```








