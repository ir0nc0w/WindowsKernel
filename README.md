# WindowsKernel

## 1.Intro
본 노트는 Windows kernel fuzzer를 조사 목적으로 작성한다. 정리는 아래와 같은 양식으로 정리한다. 

- Name
- Purpose
- Support OS
- Key ideas
- Pros and Cons
- Limitation

## 2.Syzkaller
### Purpose

### Support OS
처음에는 Linux kernel를 타겟으로 개발되었지만, 현재 다른 OS kernel 지원을 확장하였다.
OS Kernel: Linux kernel, Akaros, Darwin/XNU, FreeBSD, Fuchsia, NetBSD, OpenBSD, Windows, gVisor

### Key ideas

### Pros and Cons


### Limitation
Windows OS는 GCE(Google Computer Engine) 가상머신(VM)만  지원

## 3.WinAFL+IntelPT


## 4.ioctlfuzzer:cr4sh
### Purpose
Searching vulnerabilities in Windowss kernel driver 

### Support OS
Windows XP, Vista, 2003 server, 2008 server, 7 (32bit and 64bit)

### Key ideas
1) Hooking and interception
  The fuzzer's own driver hooks NtDeviceControlFile in order to take control of all IOCTL requests throughout the system.
  
__kernel_entry NTSTATUS NtDeviceIoControlFile(
  IN HANDLE            FileHandle,
  IN HANDLE            Event,
  IN PIO_APC_ROUTINE   ApcRoutine,
  IN PVOID             ApcContext,
  OUT PIO_STATUS_BLOCK IoStatusBlock,
  IN ULONG             IoControlCode,
  IN PVOID             InputBuffer,
  IN ULONG             InputBufferLength,
  OUT PVOID            OutputBuffer,
  IN ULONG             OutputBufferLength
);
  
## 5.IOCTLbf


## 6.kAFL

## 7.PTFuzz


