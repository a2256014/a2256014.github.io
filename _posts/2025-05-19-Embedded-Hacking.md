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

| **이름**  | **역할**                                   |
| ------- | ---------------------------------------- |
| r13(SP) | Stack Pointer, x86-64에서의 rsp             |
| r14(LR) | Link Register, return 주소 저장              |
| r15(PC) | Program Counter, x86-64에서의 rip           |
| CPSR    | Current Program Status Register, 상태 레지스터 |

#### CPSR 레지스터

- **N**egative flag : 결과값이 음수일 때 1로 설정
- **Z**ero flag : 결과값이 0일 때 1로 설정
- **C**arry flag : 덧셈에서 Carry 발생 시 1로 설정, 뺄셈에서 Borrow 발생 시 0으로 설정
- o**V**erflow flag : 덧셈 / 뺄셈에서 Signed Overflow가 발생 시 1로 설정
- 7~6번 비트 : IRQ와 FIQ를 비활성화 하는 비트로 1로 설정 시 비활성화
- 5번 비트 : Arm, Thumb 모드를 나타냄 0:Arm, 1:Thumb
- 4~0번 비트 : 프로세서 모드를 나타냄

|**M[4:0]**|**Mode**|**역할**|
|---|---|---|
|0b10000|User|애플리케이션이 동작하는 모드, CPSR에서 flag field만 수정 가능|
|0b10001|Fast Interrupt Request (FIQ)|높은(fast) 우선순위 인터럽트 발생 시 진입|
|0b10010|Interrupt Request (IRQ)|낮은 우선순위 인터럽트 발생 시 진입|
|0b10011|Supervisor (SVC)|시스템 콜(SVC) 발생 시 진입|
|0b10111|Abort|메모리 접근 실패 시 진입|
|0b11011|Undefined|정의 되지 않은 명령어 확인 시 진입|
|0b11111|System|User 모드의 Privileged 버전, CPSR의 모든 비트 수정 가능<br><br>(User 모드와 레지스터 공유)|

![](/assets/images/posts/2025-05-19-Embedded-Hacking/0f1ada733c4607b94f95cd729fc62fc7_MD5.jpeg)


#### Banked 레지스터
banked register는 각 모드 별로 물리적으로 존재하는 레지스터이며 banked register가 아닌 레지스터들은 `User & System` 모드의 레지스터를 공유한다.
공유하는 레지스터는 모드 변경 시 기존 모드의 레지스터 값을 스택에 넣어 현재 컨텍스트를 저장한다.

![드림핵 출처](/assets/images/posts/2025-05-19-Embedded-Hacking/548c86d6914a1d15a8a9a31bceee3573_MD5.jpeg)

### ARM 어셈블리
#### 기본 구조 및 명령어
> 각 명령어는 공부하면서 차근차근 익히자

```assembly
명령어{s}{condition} Rd, Rn, {Operand2}
```

| **명령어**                              | **형식**                 | **설명**                                     |
| ------------------------------------ | ---------------------- | ------------------------------------------ |
| `MOV`                                | MOV Rd, Rn, {Operand2} | Rn을 Rd에 넣는다.                               |
| `MVN`                                | MVN Rd, Rn, {Operand2} | Rn의 NOT을 Rd에 넣는다.                          |
| `LDR`                                | LDR Rd, {Operand2}     | Operand2에 있는 값을 Rd에 넣는다.                   |
| `STR`                                | STR Rd, {Operand2}     | Rd에 있는 값을 Operand2에 넣는다.                   |
| `ADD`                                | ADD Rd, Rn, Operand2   | Rn + Operand2를 Rd에 넣는다.                    |
| `SUB`                                | SUB Rd, Rn, Operand2   | Rn - Operand2를 Rd에 넣는다.                    |
| `MUL`                                | MUL Rd, Rn, Operand2   | Rn * Operand2를 Rd에 넣는다.                    |
| `AND`                                | AND Rd, Rn, Operand2   | Rn & Operand2를 Rd에 넣는다.                    |
| `ORR`                                | ORR Rd, Rn, Operand2   | Rn \| Operand2를 Rd에 넣는다.                   |
| `EOR`                                | EOR Rd, Rn, Operand2   | Rn ^ Operand2를 Rd에 넣는다.                    |
| `BIC`                                | BIC Rd, Rn, Operand2   | Rn & ~Operand2를 Rd에 넣는다.                   |
| `CMP`                                | CMP Rn, Operand2       | Rn - Operand2를 하여 CPSR의 flag를 설정한다.        |
| `CMN`                                | CMN Rn, Operand2       | Rn + Operand2를 하여 CPSR의 flag를 설정한다.        |
| `B`(Branch)                          | B \<target address\>   | target address로 분기한다.                      |
| `BL`(Branch with Link)               | BL \<target address\>  | 다음 명령어의 주소를 LR에 저장하고 target address로 분기한다. |
| `BX`(Branch and Exchange)            | BX \<Rm\>              | Rm에 저장된 주소로 분기한다.                          |
| `BLX`(Branch with Link and Exchange) | BLX \<Rm\>             | 다음 명령어의 주소를 LR에 저장하고 Rm에 저장된 주소로 분기한다.     |
| `PUSH`                               | PUSH \<registers\>     | 주어진 레지스터들을 스택에 push 합니다.                   |
| `POP`                                | POP \<registers\>      | 주어진 레지스터들을 스택에서 pop 합니다.                   |
| `SVC`                                | SVC \<immediate\>      | immediate 값에 해당하는 소프트웨어 인터럽트를 발생시킵니다.      |

#### 프롤로그
> x86-64는 `call`이지만, arm은 `bl blx` 명령어 실행 시 LR에 반환 주소가 저장된다.


```assembly
# Prologue
push {fp, lr}       - LR FP 순으로 Stack에 저장
add fp, sp, #4      - FP를 SP + 4 로 이동
sub sp, sp, #12     - SP를 SP - 12 로 이동

...
```

#### 에필로그
```assembly
...

# Epilogue
sub sp, fp, #4      - SP를 FP - 4 로 이동
pop {fp, pc}        - FP PC 순으로 레지스터 값 pop
```

### 환경 구축
#### QEMU 설치
> Linux x86-64환경에서 Arm 바이너리를 실행시키기 위한 에뮬레이터 -> 즉, 맥북만 있으면 만사 OK.....

```shell
sudo apt-get update
sudo apt-get install qemu-user-static libc6-armel-cross gdb-multiarch -y

qemu-arm-static -version

# gdb 붙이기 위한 port open
qemu-arm-static -g [port] [binary path]
# gdb 실행
gdb-multiarch [binary path]
# 붙기
target remote :[port]

# 에러 발생 시
export QEMU_LD_PREFIX=/usr/arm-linux-gnueabi
```

#### Ghidra 설치
> 이건 너무 많이 해서 넘김 - 나중에 노션에 정리한거 가져오기

### ARM 공격 기조
>공격 방식이 X86-64와 달라진 건 레지스터의 명밖에 없다고 생각하면 편하다.

X86-64에서 BOF 시 리턴 주소를 가젯으로 덮어 쓰면서 이어갔는데, ARM에서도 마찬가지다 단지, ARM에서는 리턴 주소가 `lr` 레지스터에 있다. [ARM 레지스터](##레지스터)

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
> 부트로더, 커널, 파일시스템 존재하며, 하나의 데이터로 구성되어 있기에 어디부터 어디까지가 어느 부분인지 경계가 모호하다.

#### Binwalk
>경계가 모호한 펌웨어에 대해서 시그니처를 식별하여 부트로더, 커널, 파일시스템의 영역을 확인할 수 있는 툴이다.

```shell
sudo apt install binwalk
binwalk ./Target_Firmware.bin

# 데이터 추출
binwalk -e ./Target_Firmware.bin --run-as=root

# 에러 발생 시
pip uninstall capstone
pip install capstone==3.0.5

# Flattened device tree 추출
dd if=./Target_Firmware.bin skip=2983576 bs=1 count=489832 of=./aa.bin
fdtdump ./aa.bin
```

![](/assets/images/posts/2025-05-19-Embedded-Hacking/5592782f0e696c43aca8936092628194_MD5.jpeg)

부트로더인 `uboot.bin`, 커널 데이터 `54798.7z` 파일의 압축을 푼 `54798` 그리고 squash 파일 시스템인 `squashfs-root`가 들어있다.
![](/assets/images/posts/2025-05-19-Embedded-Hacking/2440894647f15f13f19079fdd6efbba1_MD5.jpeg)

![](/assets/images/posts/2025-05-19-Embedded-Hacking/bd8bd24d233064b86b4ee9df69469521_MD5.jpeg)

#### 부트로더 확인
>`uboot.bin`파일은 부트로더가 작동할 때 필요한 데이터 및 문자열이 들어있다.

```shell
strings ./uboot.bin
```

| **U-boot Env** | **설명**                                                                                                                                               |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| bootargs       | 부팅할 때 필요한 명령어의 매개 변수                                                                                                                                 |
| bootcmd        | bootdelay 초 동안 사용자가 아무것도 입력하지 않았을 때 자동으로 실행되는 명령어                                                                                                    |
| bootdelay      | bootcmd를 실행하기 전의 딜레이 시간(초), 이 시간 동안 사용자 입력 가능<br>- 0 : delay 없음, 하지만 사용자 입력으로 멈출 수 있다.<br>- -1 : autoboot 비활성화<br>- -2 : delay 없음, 사용자 입력으로 멈출 수 없다. |
| baudrate       | UART의 baudrate를 설정                                                                                                                                   |
![](assets/images/posts/2025-05-19-Embedded-Hacking/8eddc143ebf0a9bba9d89c316737e05c_MD5.jpeg)

#### Appendix - 펌웨어 보호
>파일 시스템을 암호화하고 부트로더 및 커널에 복호화 코드를 넣는 방식으로 파일 시스템을 보호한다.

- Chain of Trust : 하드웨어와 소프트웨어 모두 안전한 상태로 구축하기 위한 구조
- Root of Trust : 절대 변조되거나 복제할 수 없는 첫 부분 (예로 실생활에서는 지문이 있다.)

임베디드의 경우 RoT는 FPGA칩이 사용된다.
1. RoT에 있는 Key를 가지고 첫 번째로 실행할 코드(ex. Boot Loader)를 암호화(ex. AES) 해서 넣는다.
2. 첫 번째 코드의 내부에 가지고 있는 키를 가지고 두 번째 코드(ex. Kernel)를 암호화 한다.
3. Application을 실행 시키는 단계까지 위와 같은 과정을 거친다.
4. 이후 기기가 작동하면, RoT를 기점으로 다음 Stage를 복호화 하면서 코드를 실행한다



### 펌웨어 에뮬레이션

### 펌웨어 디버깅 환경 구축

### 펌웨어 공격 실습