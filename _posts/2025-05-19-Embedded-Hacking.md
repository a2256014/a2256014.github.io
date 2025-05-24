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

#### Squash 파일 시스템
> UBIFS, Cramfs 등 파일 시스템 종류는 다양하다.

Squash 파일 시스템의 구성도 / [squashfs 공식](https://dr-emann.github.io/squashfs/)

| **이름**                          | **역할**                                                 |
| ------------------------------- | ------------------------------------------------------ |
| Superblock                      | 다른 섹션의 위치를 포함한 아카이브의 중요한 정보가 저장된다.                     |
| Compression Options             | `COMPRESSOR_OPTIONS` flag가 세팅되면 압축 옵션들이 해당 섹션에 저장된다.   |
| Datablocks & Fragments          | 아카이브 내에 저장된 파일의 데이터가 블록 단위로 해당 섹션에 저장된다.               |
| Inode Table                     | 아카이브 내에 저장된 파일의 메타데이터(소유권, 권한 등)가 저장된다.                |
| Directory Table                 | 파일 이름과 inode에 대한 참조를 포함한 디렉토리 목록이 저장된다.                |
| Fragment Table                  | `Datablocks & Fragments` 섹션 내에 위치한 Fragment의 설명이 저장된다. |
| Export Table                    | NFS export에 필요한 inode 번호에서 디스크 위치로의 매핑이 저장된다.          |
| UID/GID Lookup Table            | UID와 GID의 리스트가 저장된다. ID들은 이 테이블을 통해 인덱스로 참조된다.         |
| Xattr(Extended attribute) Table | 아카이브 내 항목들의 확장 속성이 저장된다.                               |

```shell
# https://github.com/sbourdelin/squashfs-info
make

./squashfs-info ./350000.squashfs
```

| **이름**                | **설명**                                              |
| --------------------- | --------------------------------------------------- |
| s_magic               | 문자열로 "hsqs"(little)의 magic 값                        |
| inodes                | inode 개수                                            |
| mkfs_time             | 만들어진 시간                                             |
| block_size            | block 1개 크기                                         |
| fragments             | fragment table 항목 개수                                |
| compresultsion        | 압축 유형(4의 경우 xz)                                     |
| block_log             | block_size의 이진 로그 값                                 |
| flags                 | Superblock에 대한 flag                                 |
| no_ids                | id look table의 항목 수                                 |
| s_major               | squashfs 파일 포맷의 major 버전(4이어야 함)                    |
| s_minor               | squashfs 파일 포맷의 major 버전(0이어야 함)                    |
| root_inode            | archive 내 root directory의 inode                     |
| bytes_used            | 아카이브 내에서 사용된 바이트                                    |
| id_table_start        | id table 시작 바이트 오프셋                                 |
| xattr_id_table_start  | xattr id table 시작 바이트 오프셋(table이 없다면 0xfffffff로 설정) |
| inode_table_start     | inode table 시작 바이트 오프셋                              |
| directory_table_start | directory table 시작 바이트 오프셋                          |
| fragment_table_start  | fragment table 시작 바이트 오프셋                           |
| lookup_table_start    | export table 시작 바이트 오프셋                             |
![](/assets/images/posts/2025-05-19-Embedded-Hacking/226646ecfcf1bf6e0895a628fc885843_MD5.jpeg)
#### 파일 시스템 분석
> 리눅스 부팅 과정 : /linuxrc -> /etc/inittab -> /etc/init.d/rcS

![](/assets/images/posts/2025-05-19-Embedded-Hacking/cb1a025f1d5f7c5d617dcef2fc191117_MD5.jpeg)

1. `linuxrc`는 부팅 시 처음 실행되는 바이너리로 `/etc/inittab` 설정 파일을 읽는다.
   `squashfs-root` 디렉토리에서 서비스 바이너리 찾기
   ![](/assets/images/posts/2025-05-19-Embedded-Hacking/110767eadbd0e951e5c4c7192de9b56b_MD5.jpeg)
2. `/etc/inittab` 파일은 부팅 시 필요한 정보를 담는 파일이다.
   inittab 파일의 작성 포맷은 `<id>:<runlevels>:<action>:<process>` 구조다.
	- id : 프로세스를 실행할 tty
	- runlevels : runlevel 선택, 하지만 busybox의 init에서는 지원 X
	- action : 프로세스를 어떻게 다룰 것인지 선택
	    - respawn : 만약 작성된 프로세스가 진행 중이라면 실행 X, 진행 중이 아니라면 실행, 중간에 죽으면 다시 실행
	    - wait : 선택한 runlevel에 해당하는 runlevel이 init을 하면 프로세스 실행하고 죽기 전까지 대기
	    - once : 선택한 runlevel에 해당하는 runlevel이 init을 하면 프로세스 실행, 대기 X
	    - askfirst : respawn과 동일한 역할을 하지만, 프로세스를 실행하기 전 `"Please press Enter to activate this console."`이라고 출력
	    - sysinit : init에서 console에 접근하기 전에 실행
	    - shutdown : 기기가 종료될 때 실행
	- process : 실행할 프로그램
   ![](/assets/images/posts/2025-05-19-Embedded-Hacking/72724a4217f012e6cb4cdc503b80931c_MD5.jpeg)
3. rcS 스크립트는 부팅 시에 실행되는 쉘 스크립트이다.
   >rcS 파일은 개발자 나름이라 펌웨어마다 해당 파일에 모든 코드가 있을 수도 있고 여러개로 나뉠 수도 있다.
   
   ![](/assets/images/posts/2025-05-19-Embedded-Hacking/6a20b473c5df2785d035ee855dd2a50d_MD5.jpeg)
4. `/etc/init.d` 폴더에서 서비스 바이너리 찾기 (서비스 바이너리 위치 : `/app/app`)
   ![](/assets/images/posts/2025-05-19-Embedded-Hacking/ae91e8d88b2f3bfee419cca904621adc_MD5.jpeg)

#### 리눅스 계정 정보 확인
> 쉬운 비밀번호의 경우 `john-the-ripper`를 사용해 Brute-force 공격

계정 정보는 `/etc/passwd`, `/etc/shadow`에서 확인

#### Appendix - Busybox
>busybox로 symlink 되어 있어서 어느 명령어를 실행시키든 busybox가 실행이 되며 busybox 내에서 `argv[0]`를 참조해 어떤 명령어를 실행했는지 확인하고 이후 명령어에 맞는 코드를 실행하는 구조다.

Busybox는 주로 자원이 한정적인 임베디드 기기가 많이 사용하는 바이너리로, 유닉스 명령어들이 busybox 실행 파일 안에 모두 들어있어 명령어 실행 파일의 용량을 줄인다.

![](/assets/images/posts/2025-05-19-Embedded-Hacking/fa3967663d09f348bf7e56794be58106_MD5.jpeg)

### 펌웨어 에뮬레이션 - QEMU
**QEMU**는 X86, Arm, MIPS, PowerPC, RISC-V, s390x, SPARC 등의 아키텍처를 지원하는 오픈소스 머신 에뮬레이터로 **User-mode Emulation**과 **System Emulation**가 있다.

#### User-mode Emulation
User-mode Emulation은 다른 아키텍처의 CPU를 에뮬레이트 한다. 이를 통해 다른 아키텍처의 실행 파일을 실행할 수 있다. 또한 `-g [port]` 옵션을 사용하여 gdb-multiarch를 통한 디버깅이 가능함 
User-mode Emulation의 예시로 `qemu-arm-static` 등이 있다.

#### System Emulation
System Emulation은 보드 안에 있는 부품을 포함하여 보드 자체를 에뮬레이트 한다. 또한 보드에 다른 기기들을 추가할 수도 있다. `qemu-system-arm`에서 `-device help` 옵션을 통해 추가 가능한 기기들을 확인할 수 있다.
```shell
qemu-system-arm -device help
```

![](/assets/images/posts/2025-05-19-Embedded-Hacking/efe32fb58f6499cfb47cbf3f63748195_MD5.jpeg)

#### 사용법
>**System Emulation**을 사용하여 펌웨어를 작동 시키려면 `Linux 커널 이미지`, `파일시스템` 필요

Arm 보드로는 `Arm Versatile Board`, `Arm Versatile Express Board`, `virt`가 있고 각 보드마다 지원하는 Arm CPU가 다르기 때문에 **파일 시스템에 있는 바이너리가 어느 ISA 버전으로 빌드 됐는지** 보고 보드를 선택하면 된다.

```shell
# 설치
sudo apt install qemu-system-arm -y

# 보드 리스트 확인
qemu-system-arm -M help

# 보드 선택
qemu-system-arm -M [board name]

# Block device 연결
qemu-system-arm -drive file=[file_path],format=[type]

# RAM Disk 연결
qemu-system-arm -initrd [file_path]

# 네트워크 구성 아래 표 참고
```

| **네트워크 구성**       | **사용법**                                                                                                                                               | **설명**                                                                                                                                                                                                                                                                                                                                                                                                   |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TAP network       | qemu 실행 시 아래 옵션 추가<br><br>-netdev tap,id=[id],ifname=[ifname]<br><br>-device [device_name],netdev=[id]<br><br>---<br><br>host, guest의 tap 인터페이스 ip 설정 | `-netdev` 옵션을 사용하여 host에 가상 네트워크 인터페이스인 TAP Device를 구성 `id`에는 이후 `-device` 옵션과 연결하기 위한 임의 id를 설정하고 `ifname`에는 TAP 인터페이스의 이름을 설정<br><br>`-device` 옵션으로 guest에 가상 네트워크 기기를 추가하여 host와 실제 기기처럼 통신할 수 있다. `netdev`에는 이전에 설정한 `netdev` id를 입력하여 서로 연결<br><br>실행 시 /etc/qemu-ifup 스크립트에 의해 host에 자동으로 tap 인터페이스가 생성, 이후 host와 guest의 ip를 설정하면 서로 통신할 수 있다. 종료 시 /etc/qemu-ifdown 스크립트에 의해 host의 tap 인터페이스가 종료됨 |
| User-mode network | qemu 실행 시 아래 옵션 추가<br><br>-netdev user,id=[id],[options]<br><br>-device [device_name],netdev=[id]                                                     | `-netdev` 옵션을 사용하여 host의 network를 공유하는 가상 네트워크 인터페이스를 구성<br><br>이후 `-device` 옵션으로 guest에 가상 네트워크 기기를 추가하여 host의 네트워크를 공유할 수 있다.<br><br>특정 포트와 통신을 할 예정이라면 `hostfwd=[tcp/udp]::[port]-:[port]` 옵션을 추가                                                                                                                                                                                                     |

>그 외에도 Network File System(NFS), ssh disk image, NVMe disk image 등 많은 프로토콜과 이미지를 지원 [QEMU Docs](https://qemu-project.gitlab.io/qemu/system/images.html) 참조

#### 구축(System Emulation)

```shell
# virt 가상 보드 사용
# -nographic : CLI 환경에서는 해당 옵션 필요
qemu-system-arm \
-kernel ./zImage \
-initrd ./_Target_Firmware.bin.extracted/350000.squashfs \
-M virt \
-nographic \
-append "root=/dev/ram0" \
-netdev user,id=eth0,hostfwd=tcp::1413-:1413,hostfwd=tcp::8899-:8899 \
-device virtio-net-device,netdev=eth0
```

![](/assets/images/posts/2025-05-19-Embedded-Hacking/d527a6dfb1092d4c5ba630c159cef83a_MD5.jpeg)

### 펌웨어 디버깅 환경 구축

### 펌웨어 공격 실습