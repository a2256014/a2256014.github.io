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
## 필요 장비
> 사용법은 이름을 검색해서 찾아보자..... 너무 많다.

### 납땜 준비물
- 인두기 - 하코 FX-888D - UART 연결, JTAG 연결, eMMC 읽기
- 인두기 스탠드 - 하코 FX-888D 스탠드 - 인두기 사용 시 필요
- 펜 플럭스 - [SME] 펜 플럭스 (유연납용) - 납땜 시 필요, 디솔더링 시 필요
- 솔더링 페이스트 - BURNLEY 페이스트 - 납땜 시 필요
- 실납 - [신성금속] 실납(유연)-송진(70G, SN45) - 핀헤더 납땜, eMMC 읽기
- 납 흡입기 - [EXSO] 수동흡입기 [DS-1010] - 납땜 시 필요
- 솔더윅 - [HAKKO] 솔더위크 FR150-86 - 납땜 시 필요, UART 연결, eMMC 읽기
- 인두 팁 클리너 - 하코 FX-888D 스탠드에 있는 팁 클리너 - 납땜 시 필요
- 에나멜 동선 - [NW3 (중국)] 소용량 에나멜 동선 UEW-0.10mm - UART 연결, JTAG 연결
- eMMC Reballing Stencil - Amaoe universal emmc reballing stencil - eMMC 읽기
- Solder Paste - XeredEx Solder Paste 50g(솔더링 페이스트와 구분 중요) - eMMC 읽기

### 보드
- 아두이노 우노 - 아두이노 우노 - Voltage Glitching 실습, Timing Attack 실습
- 라즈베리파이 4B - 라즈베리파이 4B - JTAG 연결, Flash memory 읽기, Voltage Glitching 실습
- 라즈베리파이 3B+ - 라즈베리파이 3B+ - JTAG 연결

### 디솔더링 장비
- 열풍기 - 스탠리 열풍기 STEL670 - Flash memory 읽기, eMMC 읽기, 디솔더링 시 필요
- 핀셋 - ESD 코팅 정전기 방지 핀셋 - Voltage Glitching 실습, Timing Attack 실습, Flash memory 읽기, eMMC 읽기, 디솔더링 시 필요

### 디버깅 장비
- Logic Analyzer - USB Logic Analyzer 24mhz / 8 channel - UART 연결, Timing Attack 실습
  라즈베리파이와 컴퓨터의 UART 통신을 중간에서 읽기위해서 먼저 신호를 잡으려면 컴퓨터에 소프트웨어 설치를 해야한다. Logic Analyzer의 신호를 잡는데 사용할 수 있는 소프트웨어는 `Saleae Logic Analyzer`([링크](https://www.saleae.com/downloads/))와 `Sigrok PulseView`([링크](https://sigrok.org/wiki/Downloads))를 사용할 수 있습니다.
  **Windows 10** 환경으로 `Saleae Logic Analyzer`를 사용하며, 각 [OS 별 드라이버 설치](https://support.saleae.com/logic-software/driver-install)와 [기기 연결 확인](https://support.saleae.com/troubleshooting/pc-detection-test)은 링크를 통해 확인
- USB to UART 케이블 - PL2303TA usb to ttl 시리얼 케이블 - UART 연결
- 테스트 후크 클립 - Cleqee P1512D 테스트 후크 클립(해외) - UART 연결, Flash memory 읽기
- JTAG 디버거 - J-Link V9 - JTAG 연결
- 메모리 프로그래머 - Xgecu TL866II Plus 프로그래머 - Flash memory 읽기
- TSOP 48 소켓 - TL8666II Plus TSOP 48 소켓 - Flash memory 읽기

### 메모리 관련 준비물
- eMMC 메모리가 붙어있는 아무 보드 - eMMC 메모리(`KLM4G1FETE-B041`) - eMMC 읽기
- eMMC 모듈 리더 보드 - 하드커널 eMMC Module Reader Board for OS upgrade - eMMC 읽기
- eMMC 모듈 - 하드커널 8GB eMMC Module C2 Linux - eMMC 읽기
- NOR Flash 메모리가 붙어있는 아무 보드 - NOR Flash 메모리 - Flash memory 읽기
- NAND Flash 메모리가 붙어있는 아무 보드 - NAND Flash 메모리 - Flash memory 읽기

### 회로 준비물
- 브레드보드 - 브레드보드 - Voltage Glitching 실습, Timing Attack 실습
- NPN 트랜지스터 - 2N2222A 트랜지스터 - Voltage Glitching 실습
- 점퍼선(암-암, 암-수, 수-수) - 브레드보드 점퍼선 - UART 읽기, JTAG 읽기, Voltage Glitching 실습, Timing Attack 실습
- 4.7K옴 저항기 - 4.7K옴 저항기 - Voltage Glitching 실습
### 기타 준비물
- 핀헤더 - 핀헤더 - UART 읽기
- 장갑 - 3M 슈퍼그립 - 납땜 및 모든 작업 시 필요
- 드라이버 - 샤오미 미지아 정밀 드라이버 세트 - 장비 분해 시 필요
- 니퍼 - 니퍼 - 장비 분해 시 필요
- 멀티미터 - 플루크 FLUKE-101 - UART 읽기, eMMC 읽기
- 내열 실리콘 작업 패드 - COMS 내열 실리콘 작업 패드 - 디솔더링 시 필요
- SD 카드 리더기 - 트랜센드 TS-RDF5 USB3.1 카드리더기 - eMMC 읽기

## 디버그 인터페이스
> 디버그 인터페이스를 사용하여 펌웨어 추출 진행!!

### UART
**UART (Universal Asynchronous Receiver-Transmitter)** 는 장치들간의 비동기 직렬 통신을 위한 인터페이스이다.

클럭 핀은 1비트의 데이터를 받는다고 가정했을 때, 어디까지가 1비트인지 비트의 시작과 끝을 정의하는데, 비동기 통신이기 때문에 클럭 핀이 존재하지 않는다. 

클럭 핀을 대신하여 **Baud rate** 를 사용하는데, 전송 속도를 나타내는 단위이다.

1. UART 핀 찾기
2. 쓰루홀 핀[납땜을 통해] / 패드[에나멜 동선을 통해] 상황에 맞게 핀헤더 연결하기
3. 

### Appendix

### JTAG