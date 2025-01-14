# Implementation of FlexSC on Linux Kernel v5.0+ and Performance Analysis

## FlexSC
FlexSC (Flexible System Call), a mechanism to process system call, which was introduced on [OSDI'10](https://www.usenix.org/conference/osdi10/flexsc-flexible-system-call-scheduling-exception-less-system-calls) by Livio Soares.

The main concept of FlexSC is processing syscalls in batching way, which has better cache locality and almost no CPU mode switch involved. For more details, the link above provides link to the paper. Also, you can refer to my porting note at [HackMD](https://hackmd.io/@foxhoundsk/S1Wdf-g0V).

## How Syscall Being Processed by FlexSC
Syscalls are processed through the following steps:

1. The moment syscall being requested by user thread, it simply grab a free syscall entry, and submit (change the state of the entry) it after done population of syscall-related arguments to the entry
2. Once there are no free entries, the kernel visible thread start submitting (by marking syscall entry to different state) the entries to kthread
3. Kthread detects that it got stuff to do (by scanning syscall entries), then it start queuing work to the CMWQ workqueue
4. After the work (syscall) is done, kthread change the state of the syscall entry
5. Library of FlexSC (user space) detects that the syscall is done, it simply return the retval of the syscall to application thread

Illustration of FlexSC:
```
        +---------------------------+
        |                           |
        |   user thread requesting  | .....
        |          syscalls         |
        |                           |
        +---------------------------+

        +---------------------------+
        |                           |
        |   kernel-visible thread   |
        |                           |                   +-----------+
        +---------------------------+                   |           |
                                        USER SPACE      |  shared   |
--------------------------------------------------------|  syscall  | .....
                                       KERNEL SPACE     |   entry   |
        +---------------------------+                   |           |
        |                           |                   +-----------+
        |   kthreads dispatching    |
        |  work to CMWQ workqueue   |
        |                           |
        +---------------------------+
```

## Implementation
The repo was originally downloaded from splasky/flexsc ([c69213](https://github.com/splasky/linux/tree/c69213aabcb1b6046ade5dbacfc95d1d0356ea14)), it was lacking many of implementation of FlexSC at that commit, and what I've implemented are the following:

- per-kthread syscall entries
- kernel-visible thread (pthread)
- performance measurement program (write() and getpid() syscall)
- func of kthread
- mechanism to get free syscall entry
- allocation of CMWQ (Concurrency Managed Workqueue) and its work

**Currently**, `flexsc.c` at `libflexsc/` and `linux-5.0.10/flexsc/` having some hard-coded section which used to test `write()` syscall, a clearer version can be found at [here](https://github.com/flawless0714/FlexSC/blob/master/libflexsc/versions/per_kthread_getpid/flexsc.c) (library) and [here](https://github.com/flawless0714/FlexSC/blob/master/linux-5.0.10/flexsc/versions/per_kthread_getpid/flexsc.c) (kernel code).


## Analysis
The following analysis are done with 7 kthreads (kernel cpu, each kthread handling its own works of CMWQ) and 1 kernel-visible thread (user cpu) on 8th-gen Intel CPU (i5-8350U) with HyperThreading enabled (4C8T).

- write() syscall:

    - Execution time:                 
    ![Screen](./libflexsc/perf_result/write.png)
    
    - Time elapsed for finding marked syscall entry (starts from the time the entry being marked as `FLEXSC_STATUS_MARKED`): 
    ![Screen](./libflexsc/perf_result/find_marked_syspage_elapsed_time.png)
    
    (you may figure that `thread no.` of FlexSC has ~2xx offset compare to the normal one, they have same meaning actually. It's because of I use `gettid()` for thread no. instead of order of thread creation which normal one uses.)

    - Time elapsed for library of FlexSC to find free syscall entry:
    ![Screen](./libflexsc/perf_result/get_free_syspage_elapsed_time.png)

    Summing analysis above, we might only optimize FlexSC to having similar result as typical syscall mechanism in the end, because processing syscall (write) in CMWQ costs ~500ns, summing it with other costs might lead to same consequence as I just mentioned.

- getpid() syscall:
    - Execution time:           
     ![Screen](./libflexsc/perf_result/getpid.png)

### Conclusion
It's been 10 years since FlexSC released, computer organization may changed a lot (e.g. [CPU mode switch in modern processor takes only <50ns within a round trip](https://i.imgur.com/bfgu0EK.png)). Therefore, even FlexSC doesn't has better performance than typical syscall, this is still a record which shows that imporvements of cache locality and mode switch can't still beats the time cost of typical syscall. Or, there exists some overheads within my implementation of FlexSC, feel free to open a issue if you find out anything. Thank you!

## Usage
For library, you can `$ make` directly after you've done modification of the code, I have pre-defined macros in the Makefile, hence single `$ make` command will produce two executables for testing typical syscall and FlexSC syscall respectively.

For FlexSC flavored kernel, I've added `.config` file for kernel build, you can use `$ make bzImage` to compile the kernel directly if your machine have already meet the requirement of compiling Linux kernel.

## TODO 
- (proposed by @afcidk) Find a policy to make user process to sleep during processing of requested syscalls, and wake up the process (maybe by signal) once the process is done. This is in order to reduce time elapsed by processing syscalls. Since user process  is busy-waiting (by using `pthread_yield()`, we've tried `pthread_cond_wait()`, but it's worser than `pthread_yield()`, might caused by my implementation) for the result of syscall, we want to test if putting them to sleep shows better performance. Moreover, I've noticed that syscall page (we have 64 * 7-cpu => 448) are not all exhausted by user process, this should also take into the consideration.

## Known Issues
- On exit of test program (calling `flexsc_exit()`), oops and panic will occur since the cleanup is not done correctly. It's harmless to the test since the result file is forced to flush before calling of exit of FlexSC. The workround is restart the QEMU if you are running on QEMU.

## Acknowledgement
- @[afcidk](https://github.com/afcidk) - Discussing implementation of FlexSC
- @Livio Soares - Giving such concept to execute syscall
- @[splasky](https://github.com/splasky) - Providing prototype of FlexSC
- @[jserv](https://github.com/jserv) - Giving consultancy of FlexSC
