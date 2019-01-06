  ref : https://github.com/hardenedlinux/Debian-GNU-Linux-Profiles/blob/master/docs/harbian_qa/fuzz_testing/syz_analysis.md
  This is copied from above reference and added some info about syz-manager.
  
  ### Syzkaller fuzzer
  This documentation will introduce some implement about syzkaller.
  
  1. Show you how syz-manager works
  1-1. syz-manager HTTP (User interaction)
  1-2. syz-manager SSH (syz-fuzzer in VM interaction)
  1-3. syz-manager RPC (interaction b/w syz-manager & syz-fuzzer)
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
/* Write corpus metadata into file */
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
    
    for crash := range pendingRepro {
        if reproducing[crash.Title] {
            continue
        }
        delete(pendingRepro, crash)
        if !mgr.needRepro(crash) {
            continue
        }
        log.Logf(1, "loop:add to repro queue '%v'", crash.Title)
        reproducing[crash.Title] = true
        reproQueue = append(reproQueue, crash)
    }    
    
    log.Logf(1, "loop: phase=%v shutdown=%v instances=%v/%v %+v repro: pending=%v reproducing=%v queued=%v",
        phase, shutdown == nil, len(instances), vmCount, instances,
        len(pendingRepro), len(reproducing), len(reproQueue))
    
    canRepro := func() bool {
        return phase >= phaseTriagedHub && 
            len(reproQueue) != 0 && reproInstances+instancesPerRepro <= vmCount
    }
    
    /* if crash be occured, run the repro.Run(...) func. else, run the mgr.runInstance(idx) func*/
    if shutdonw != nil {
        for canRepro() && len(instances) >= instancesPreRepro {
            last := len(reproQueue) - 1
            crash := reproQueue[last]
            reproQueue[last] = nil
            reproQueue = reproQueue[:last]
            vmIndexes := append([]int{}, instances[len(instances)-instancesPerRepro:]...)
            instances = instances[:len(instances)-instancesPerRepro]
            reproInstances += instancesPerRepro
            atomic.AddUint32(&mgr.numReproducing, 1)
            log.Logf(1, "loop: starting repro of '%v' on instances %+v", crash.Title, vmIndexes)
            go func() {
                res, stats, err := repro.Run(crash.Output, mgr.cfg, mgr.reporter, mgr.vmPool, vmIndexes)
                reproDone <- &ReproResult{vmIndexes, crash.Title, res, stats, err, crash.hub}
            }()
        }
        for !canRepro() && len(instances) != 0 {
            lastr := len(instances) - 1
            idx := instances[last]
            instances = instances[:last]
            log.Logf(1, loop: starting instance %v", idx)
            go func() {
                crash, err := mgr.runInstance(idx)
                runDone <- &RunResult{idx, crash, err}
            }()
        }
    }

```
Above procedures, we can first summarize the syz-manager control flow.
mgr.RunManager() --> mgr.vmLoop() --> 1) mgr.runInstance() or 2) repro.Run(...)

Firstly, we see mgr.runInstance(idx) :
``` go
/**/
fwAddr, err := inst.Forward(mgr.port)
...
/**/
fuzzerBin, err := inst.Copy(mgr.cfg.SyzFuzzerBin)
...
/**/
executorBin, err := inst.Copy(mgr.SyzExecutorBin)
...

// Run the fuzzer binary.
...
cmd := instance.FuzzerCmd(fuzzerBin, executorBin, fmt.Sprintf("vm-%v", index),
    mgr.cfg.TargetOs, mgr.cfg.TargetArch, fwdAddr, mgr.cfg.Sandbox, procs, fuzzerV,
    mgr.cfg.Cover, *flagDebug, false, false)
outc, errc, err := inst.Run(time.Hour, mgr.vmStop, cmd)
...
// Monitor Execution
rep := inst.MonitorExecution(outc, errc, mgr.reporter, vm.ExitTimeout)

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


