# WindowsKernel

## Intro
본 노트는 Windows kernel fuzzer를 조사 목적으로 작성한다. 정리는 아래와 같은 양식으로 정리한다. 

* Name
* Purpose
* Support OS
* Key ideas

## Syzkaller
### Purpose

### Support OS
처음에는 Linux kernel를 타겟으로 개발되었지만, 현재 다른 OS kernel 지원을 확장하였다.
OS Kernel: Linux kernel, Akaros, Darwin/XNU, FreeBSD, Fuchsia, NetBSD, OpenBSD, Windows, gVisor

### Key ideas


### Limitation
Windows OS는 GCE(Google Computer Engine) 가상머신(VM)만  지원

## KernelFuzzer
### Purpose
Windows system API fuzzer

### Support OS
Windows 

### Features
- Decoupled components
  * Knowledge Bases : OS API Knowledge Base, System Calls Knowledge Base
  * Object Store : Keep track of handles to various objects
  * Helper Functions : Generate, populate and return valid structures
  * Fuzzed Value Generator : Functions return fuzzed basic data types (ex. bool, int, float, etc)
  
### Key ideas
- 

## WinAFL+IntelPT (yet uploaded)
### Purpose

### Support OS

### Key ideas
1) In-memory fuzzing (Persistence Fuzzing Mode) : WinAFL (without Intel PT)
  Windows does not use COW(Copy-on-Write) and therefore fork-like mechanisms are not efficient on Windows
  - Persistence is key
    * Restart process each time (disable persistence) ~2.3 exec/s
    * Persist   100 iterations before restart ~72 exec/s
    * Persist  1000 iterations ~123 exec/s
    * Persist 10000 iterations ~133 exec/s
  
2) Intel PT tracing
  - Support control flow tracing
   * Decoder can determine exact flow of software execution from trace log
   * Target < 5% performance overhead
  - The Intel PT log does not contain Block IDs or all branch targets
  - Parsing large compressed logs is time consuming
  - Native persistence mode is not yet implemented
    : Work in progress using AVrf as hooking engine
  - We can filter up to 4 address ranges or whole proess

## ioctlfuzzer:cr4sh
### Purpose
Searching vulnerabilities in Windowss kernel driver 

### Support OS
Windows XP, Vista, 2003 server, 2008 server, 7 (32bit and 64bit)

### Key ideas
1) Hooking and interception
  The fuzzer's own driver hooks NtDeviceControlFile in order to take control of all IOCTL requests throughout the system.

2) Random syscall arguments or ioctl input
  IOCTL을 처리하는 동안, fuzzer는 configuration file에 지정된 조건을 준수하는 IOCTL을 스푸핑한다.
  스푸핑된 IOCTL은 입력 데이터를 제외하고 모든 점에서 원본 IRP와 동일하다. 이 데이터는 무작위로 생성된 퍼즈로 변경된다. 
 
## kAFL

### Windows
NTFS driver of Windows


