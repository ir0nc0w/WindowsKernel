  ref : https://github.com/hardenedlinux/Debian-GNU-Linux-Profiles/blob/master/docs/harbian_qa/fuzz_testing/syz_analysis.md
  This is copied from above reference and added some info about syz-manager.
  
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
In the fuzzer side, syz-fuzzer will run executors and send data to it by using pipe. In syz-fuzzer/fuzzer.go function main, we can see: 

``` go
/* flagProcs is from -procs */
for pid :=0; pid < *flagProcs; pid++ {
        /* initiate Proc struct */
        proc, err := newProc(fuzzer, pid)
        if err != nil {
                Fatalf("failed to create proc: %v", err)
        }
        fuzzer.procs = append(fuzzer.procs, proc)
        go proc.loop()
}

```
The loop() is in syz-fuzzer/proc.go. 'Generate' and 'Mutate' will determine syscall sequencee of userspace process.
'proc.execute' send data to executor to run syscalls.

e
