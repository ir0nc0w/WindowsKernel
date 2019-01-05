  ref : https://github.com/hardenedlinux/Debian-GNU-Linux-Profiles/blob/master/docs/harbian_qa/fuzz_testing/syz_analysis.md
  
  ### Syzkaller fuzzer
  This documentation will introduce some implement about syzkaller.
  
  1. Show you how fuzzer send data to executor and how executor execute a syscall.
  2. The sequence of syscalls generation
  3. User program minimize
  4. About corpus
  5. About Kcov
  
  ##Send progData to executor
  This section is about syz-fuzzer. Syz-manager will run the command in VM by ssh:
 ```
  /syz-fuzzer -executor=
 ```
