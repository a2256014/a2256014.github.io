---
layout: post
title: Hardware Hacking
subtitle: 차량 보안을 위한 기초 지식 정리 1
author: g3rm
categories: 
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - Hardware
  - Mobility
sidebar:
---


## Intro
하드웨어 해킹이란, 아래와 같이 PCB(인쇄 회로 기판)을 분석하고 공격할 수 있는 부분을 찾아내는 것이다.
![](/assets/images/posts/2025-05-12-HardwareHack/d7b29c2c03f20089dbcd02a80f887446_MD5.jpeg)

### 사례
#### 닌텐도 스위치 해킹
> 콜드 월렛 해킹, Playstation 4 Syscon 등의 해킹 사례도 같이 알아두자.

닌텐도 스위치가 사용하는 **NVIDIA Tegra X1** 칩의 **부트 롬(Boot ROM)** 코드에 취약점이 발견되었다.   
부팅과정에서 가장 먼저 실행되는 부트 롬에 위치한 프로그램이 특정 조건에 따라 복구 모드로 진입하게 되면서 해당 복구 모드의 소프트웨어에서 취약점이 발생했다.

**여기서 하드웨어 해킹은** 부트롬 코드를 읽으려고 할 때, 사용되었는데, 해당 메모리가 잠겨있어 접근이 불가능한 상태를 **Fault Injection** 공격 기법 중 하나인 Voltage Glitching을 활용하는 것이였다.

## PCB
> +극과 -극은 Power와 Ground로도 말하는데, DataSheet에서는 `-`대신 Ground/GND 라고 적혀있으니 인지하자.

> 저항기, 축전기, 유도자, 발진기, 트랜지스터, 다이오드 등 다양한 하드웨어 부품들에 대해서도 간략하게 인지하자.

**Printed Circuit Board (PCB)** 는 회로가 인쇄되어 있는 보드로 칩과 칩끼리 통신 시 직접적으로 전선을 사용하지 않고, 보드 내부 작은 구리 선을 배치하는 구조이다.

### PCB 구조

PCB를 제작하려면 먼저 설계 과정이 필요하며, PCB 설계는 EasyEDA, KiCAD 등 여러 소프트웨어를 사용한다.

1. 먼저 회로도를 설계하고
![](assets/images/posts/2025-05-12-HardwareHack/3503c5197c741fa29c82f55435ccaf2d_MD5.jpeg)

2. PCB안에 설계한 회로를 넣고 배선한다.
PCB 구조는 하나의 건물로 비유할 수 있는데, 오른쪽에서 TopLayer, BottomLayer, TopSilkLayer를 선택할 수 있는데, **TopLayer**의 경우 건물의 맨 꼭대기 층 내부에 있는 회로들입니다. 그리고 **BottomLayer**는 건물의 맨 아래층 내부에 있는 회로들입니다. **TopSilkLayer**는 건물의 외벽이다.
![](assets/images/posts/2025-05-12-HardwareHack/b88e7e8b199ac6af0e385423b9e8a110_MD5.jpeg)

3. 


### IC