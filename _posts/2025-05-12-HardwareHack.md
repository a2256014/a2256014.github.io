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
![출처-드림핵](/assets/images/posts/2025-05-12-HardwareHack/3503c5197c741fa29c82f55435ccaf2d_MD5.jpeg)

2. PCB안에 설계한 회로를 넣고 배선한다.
PCB 구조는 하나의 건물로 비유할 수 있는데, 오른쪽에서 TopLayer, BottomLayer, TopSilkLayer를 선택할 수 있는데, **TopLayer**의 경우 건물의 맨 꼭대기 층 내부에 있는 회로이며 **BottomLayer**는 건물의 맨 아래층 내부에 있는 회로이고 마지막으로 **TopSilkLayer**는 건물의 외벽이라고 비유할 수 있다.
![출처-드림핵](/assets/images/posts/2025-05-12-HardwareHack/b88e7e8b199ac6af0e385423b9e8a110_MD5.jpeg)

> PCB 층간 통신은 작은 구멍과 큰 구멍으로 나와있는 곳을 통해 통신을 진행한다. 작은홀의 경우 **비아홀(Via Hole)** 이 사용되며, 큰 홀의 경우 **PTH(Plated-Through-Hole)** 와 **NPTH(Non-Plated-Through-Hole)** 가 있다.

### IC
> http://www.visual6502.org/sim/varm/armgl.html - ARM1 구조와 회로 작동 모습 확인

IC는 **집적 회로(Integrated Circuit)** 를 의미하며, PCB에 들어가는 칩들이다. 여러 트랜지스터를 사용하여 게이트를 만들고, 게이트를 여러 개 이어서 CPU, 메모리 등 여러 칩들을 만든다.

PCB에 사용되는 IC는 아래와 같은 것들이 있다.
- Flash Memory
- Processor
- RAM
![출처 - 드림핵](/assets/images/posts/2025-05-12-HardwareHack/1477ddd7ffdab4bfcf127021fbf43e26_MD5.jpeg)

#### Flash Memory [펌웨어 저장위치]
> 펌웨어란 장치의 작동에 필요한 데이터가 저장된 곳으로 예를 들어 카메라의 펌웨어에는 버튼을 누르면 사진을 찍는 프로그램이 포함되어 있다.

데이터를 저장하는 비휘발성 메모리로 **펌웨어** 가 저장된다.

#### RAM
> 콜드 월렛 해킹 사례에서 디버그 인터페이스에서 RAM을 읽어 패스워드를 얻는 등 중요 정보가 저장될 가능성이 농후하다.

**Random Access Memory (RAM)** 은 전력이 들어올 때만 데이터를 저장하는 휘발성 메모리로 Flash Memory보다 접근 속도가 빠르기 때문에 기기에서 실행되는 프로세스는 거의 RAM에 저장된다.

따라서, RAM을 집중적으로 봐야한다. 

#### Processor
> ARM 기반의 STM32F0 시리즈 프로세서는 Fault Injection 공격에 취약하며, 실제 PoC 및 환경 구축이 상세히 나와있다. ([링크](https://www.aisec.fraunhofer.de/en/FirmwareProtection.html)) 위 공격을 통해 RDP(ReaDout Protection) 값이 2로 설정되어 펌웨어 읽기가 불가능한 칩을 읽을 수 있다.

Flash Memory -> RAM -> Processor 순으로 이동하면서 마지막 Processor에서는 RAM에 담긴 명령어를 실행하는 역할을 한다.

#### IC 패키징
> 하드웨어 해킹에서 특정 IC와 외부의 통신을 핀을 잡아서 MITM을 한다.

CPU 밑을 보면 수많은 핀(PIN)이 달려있는데, 이를 통해 외부와 통신을 진행한다. 하지만, **IC 패키징** 에 따라 IC와 연결된 핀들의 생김새가 다르다.

아래와 같은 패키징 방법이 있으니 상황에 맞게 찾아보자.
- DIP 패키징
- BGA 패키징
- TSOP 패키징
- QFP 패키징
- SOP 패키징
- WSON 패키징
- QFN 패키징

#### DataSheet
> 칩의 어떤 핀이 어떤 역할을 하는지 알기 위해서는 해당 데이터시트를 참고해야 하지만, 너무 길다. [링크](https://pdf1.alldatasheet.co.kr/datasheet-pdf/view/1243792/WINBOND/W25Q128JV.html)에서 **Winbond W25Q128JV** 칩의 경우 47쪽에 있는 `Read Manufacturer / Device ID (90h)`, 0x90 명령어를 보내 디바이스 ID를 읽을 수 있다.

IC에 대한 모든 사양 및 기능을 세부적으로 제공한 것으로 아래와 같은 정보들이 있다.
- 플래시 메모리의 용량 및 지원하는 인터페이스
- IC 패키지 종류와 핀 간격, 칩 크기, 각 핀의 역할
- 읽기 속도, 몇 번 읽고 쓰기 가능한지
- 필요 전압/전류, 최대 전압/전류, 최소 전압/전류, 적정 온도
- 플래시 메모리 명령어

#### 인터페이스
두 개 이상의 시스템이 서로 통신하기 위해 사용되는 것으로 해당 인터페이스를 사용하여 어떤 데이터를 보낼 지는 기기 구성 별[키보드, 마우스, USB 등]로 다르다.

#### IC 핀 넘버링
이는 U 혹은 V자로 칩의 중앙에 파여있는 **Notch** 혹은 1번 핀에 있는 작은 구멍인 **dot**을 기준으로 번호를 매긴다.
![](/assets/images/posts/2025-05-12-HardwareHack/38eabedfe23613bab87eaea9187a4a8c_MD5.jpeg)