---
layout: post
title: Pwn Basic
subtitle: Pwnable 세팅 및 Pwngdb 단축키
author: g3rm
categories: 
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - Pwnable
  - Settings
  - Pwngdb
sidebar:
---
## Setting
```shell
Ubuntu 22.04
CTF용 우분투 세팅 (wsl)

sudo apt update -y
sudo apt install netcat vim git gcc ssh curl wget gdb sudo python3 python3-pip -y
sudo apt install libffi-dev build-essential libssl-dev libc6-i386 libc6-dbg gcc-multilib make tmux xterm -y
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install libc6:i386 -y
python3 -m pip install --upgrade pip
pip3 install unicorn keystone-engine pwntools ropgadget
sudo apt install libcapstone-dev -y
cd ~
# git clone https://github.com/longld/peda.git ~/peda
# echo "source ~/peda/peda.py" >> ~/.gdbinit
# git clone https://github.com/scwuaptx/Pwngdb.git
# cp ~/Pwngdb/.gdbinit ~/
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh
sudo apt install ruby-full -y
sudo gem install one_gadget seccomp-tools
```

