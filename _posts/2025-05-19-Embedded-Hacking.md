---
layout: post
title: Embedded Hacking
subtitle: 드림핵 Embedded Hacking 정리
author: g3rm
categories: 
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - example
  - markdown
sidebar:
---

## Embedded란
기기에 내장되어 실행되는 시스템을 임베디드 시스템이라 부르며 해당 시스템이 동작되고 있는 기기를 임베디드 기기라고 한다.

임베디드는 일반적으로 SoC/MCU, RAM, Flash memory가 결합된 보드로 구성된다.
![](/assets/images/posts/2025-05-19-Embedded-Hacking/7ed3273bb8981e3e3ede0a4a17b602b2_MD5.jpeg)

## Arm
> Arm/Thumb mode가 존재하여 32비트(Arm), 16비트(Thumb) 모드를 번갈아 사용할 수 있으며 그로 인해 코드 실행 효율과 메모리 사용량을 최적화한다.

임베디드에서 주로 사용되는 아키텍처이므로 알아보자.

### 레지스터
#### 범용 레지스터

| **이름**   | **역할**                                |
| -------- | ------------------------------------- |
| r0 ~ r3  | 함수의 매개변수와 반환 값을 담는 레지스터, scratch 레지스터 |
| r4 ~ r10 | 변수 레지스터                               |
| r11      | Frame Pointer, x86-64에서의 rbp          |
| r12      | Intra Procedural Call                 |

#### 특수 레지스터

|**이름**|**역할**|
|---|---|
|r13(SP)|Stack Pointer, x86-64에서의 rsp|
|r14(LR)|Link Register, return 주소 저장|
|r15(PC)|Program Counter, x86-64에서의 rip|
|CPSR|Current Program Status Register, 상태 레지스터|

#### CPSR 레지스터

- **N**egative flag : 결과값이 음수일 때 1로 설정
- **Z**ero flag : 결과값이 0일 때 1로 설정
- **C**arry flag : 덧셈에서 Carry 발생 시 1로 설정, 뺄셈에서 Borrow 발생 시 0으로 설정
- o**V**erflow flag : 덧셈 / 뺄셈에서 Signed Overflow가 발생 시 1로 설정
- 7~6번 비트 : IRQ

![](/assets/images/posts/2025-05-19-Embedded-Hacking/0f1ada733c4607b94f95cd729fc62fc7_MD5.jpeg)


#### s


## 펌웨어
비휘발성 메모리에 저장된 데이터를 가르키는 것이므로 **플래시 메모리의 데이터를 읽으면 펌웨어를 얻을 수 있다.**

### 부트로더
> 자주 사용되는 부트로더는 U-BOOT

기기가 켜질 때 가장 먼저 실행되는 프로그램으로 하드웨어 초기 설정 진행과 커널을 메모리에 로드하는 역할을 담당

1. 기기 하드웨어 초기 설정
2. 압축된 커널 이미지(zImage, bzImage, uImage)를 램으로 복사
3. 커널 이미지의 앞단에 있는 커널의 압축을 푸는 코드 실행

### 커널
사용자와 하드웨어 간 인터페이스 역할을 수행하여 화면 출력, 메모리 관리, 키보드 및 마우스 조작 등, 시스템의 거의 모든 요소를 관리한다.

### 파일 시스템
OS가 저장 장치에 들어있는 데이터 및 파일들을 체계적으로 관리하게 해준다.
데이터가 저장되어 있는 곳으로 저장되는 형식도 포함되어 있다.

| 파일 시스템     | 종류                               |
| ---------- | -------------------------------- |
| 디스크 파일 시스템 | ext3, ext4, FAT, NTFS, HFS, etc. |
| 플래시 파일 시스템 | ubifs, JFFS2, YAFFS, etc.        |
| 특수 파일 시스템  | Squashfs, Cramfs, etc.           |

### 펌웨어 분석

### 펌웨어 에뮬레이션

### 펌웨어 디버깅 환경 구축

### 펌웨어 공격 실습