  ref : https://github.com/hardenedlinux/Debian-GNU-Linux-Profiles/blob/master/docs/harbian_qa/fuzz_testing/syz_analysis.md
  This is copied from above reference and added some info about syz-manager.
  
  ### Syzkaller fuzzer
  This documentation will introduce some implement about syzkaller.
  
  1. Show you how syz-manager works
  2. Show you how fuzzer send data to executor and how executor execute a syscall.
  3. The sequence of syscalls generation
  4. User program minimize
  5. About corpus
  6. About Kcov
  
  
##Syz-manager
In syz-manager, we can see main function below: 
``` go
/*Get arguments from configuration file*/
func main() {
  ...
  RunManager(cfg, target, sysTarget, syscalls)
}   
```
The RunManager function is in syz-manager/manager.go. 

``` go
/* Set up workdir directory for 'crashes' */
crashdir := filepath.Join(cfg.Workdir, "crashes")
  
  ...

/* Create HTTP server */
mgr.initHTTP()
mgr.collectUsedFiles()

/* Create RPC server for fuzzers */
mgr.port, err = startRPCServer(mgr)

/* Get fuzzer infos through RPC server */
go func() {
   for lastTime := time.Now(); ; {
      time.Sleep(10 * time.Second)
      now := time.Now()
      diff := now.Sub(lastTime)
      lastTime = now
      /* Atomic execution */ : Get the informations from fuzzer
      mgr.mu.Lock() // ----------------------------------------------------------
      ...
      mgr.fuzzingTime += diff * time.Duration(atomic.LoadUint32(&mgr.numFuzzing))
      executed := mgr.stats.execTotal.get()
      crashes := mgr.stats.crashes.get()
      signal := mgr.stats.corpusSignal.get()
      
      mgr.mu.Unlock() // --------------------------------------------------------
      
      numReproducing := atomic.LoadUint32(&mgr.numReproducing)
      numFuzzing := atomic.LoadUint32(&mgr.numFuzzing)
      
      log.Logf(0, "VMs %v, executed %v, cover %v, crashes %v, repro %v, 
          numFuzzing, executed, signal, crashes, numReproducing) 
   }
}()

if *flagBench != "" {
   f, err := os.OpenFile(*flagBench, os.O_WRONLY|os.O_CREATE|os.O_EXCL, osutil.DefaultFilePerm)
   ...
   go func() {
      for {
          time.Sleep(time.Miinute)
          vals := mgr.stats.all()
          /* Atomic execution */
          mgr.mu.Lock() // ----------------------------------------------------------
          ...
          mgr.minimizeCorpus()
          vals["corpus"] = uint64(len(mgr.corpus))
          vals["uptime"] = uint64(time.Since(mgr.firstConnect)) / 1e9
          vals["fuzzing"]= uint64(mgr.fuzzingTime) / 1e9
          mgr.mu.Unlock() // --------------------------------------------------------
          
          /* MarshalIndent is like Marshal but applies Indent to format the output*/
          data, err := json.MarshalIndent(vals, "", "  ")
          ...
          if _, err := f.Write(append(data, '\n')); err != nil {
             log.Fatalf("failed to write bench data")
          }
       }
   }()
}
...
mgr.vmLoop()

```

The type of Manager(mgr) is shown as below:
``` go
/* It's in RunManager() func */
mgr := &Manager{
    cfg:              cfg,
    vmPool:           vmPool,
    target:           target,
    sysTarget:        sysTarget,
    reporter:         reporter,
    crashdir:         crashdir,
    startTime:        time.Now(),
    stats:            new(Stats),
    crashTypes:       make(map[string]bool),
    enabledSyscalls:  syscalls,
    corpus:           make(map[string]rpctype.RPCInput),
    disabledHashes:   make(map[string]struct{}),
    memoryLeakFrames: make(map[string]bool),
    fresh:            true,
    vmStop:           make(chan bool),
    hubReproQueue:    make(chan *Crash, 10),
    needMoreRepros:   make(chan chan map[string]bool),
    usedFiles:        make(map[string]time.Time),
}    
```
vmLoop() is in syz-manager/manager.go. VM runs 
``` go
...
for shutdown != nil || len(instances) != vmCount {
    mgr.mu.Lock()
    phase := mgr.phase
    mgr.mu.Unlock()


```


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


