# WindowsKernel

## Syzkaller
### Features
Unsupervised coverage-guided kernel fuzzer

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

### Key ideas : Type aware API fuzzing
- Decoupled components
  * Knowledge Bases : OS API Knowledge Base, System Calls Knowledge Base
  * Object Store : Keep track of handles to various objects
  * Helper Functions : Generate, populate and return valid structures
  * Fuzzed Value Generator : Functions return fuzzed basic data types (ex. bool, int, float, etc)
  
### Key ideas : Manual definition of generators per-type
- 생성한 API 및 syscall을 logging을 해야하는데, 

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

## Tools developed to test ioctl interfaces for Windows kernels
### ioctlfuzzer:cr4sh
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

### ioctlbf
이 도구의 장점은 캡처한 IOCTL에 의존하지 않는다는 것이다. 그러므로, 드라이버에 의해 지원되는 유효한 IOCTL 코드를 탐지할 수 있으며, 그것은 user land의 애플리케이션에 의해 자주 또는 전혀 사용되지 않는다.
ref: https://code.google.com/archive/p/ioctlbf/

### ioctlfuzz
A mutation based user mode (ring3) dumb in-memory Kernel Driver (IOCTL) Fuzzer/Logger. This script attach it self to any given user mode process and hooks DeviceIoControl!Kernel32 API call and try to log or fuzz all I/O Control code I/O buffer length that user process sends to any Kernel driver.

ref: https://github.com/debasishm89/iofuzz

## kAFL

### Windows
NTFS driver of Windows


