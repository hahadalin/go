# sysmon - 系统监控

`sysmon`函数在proc.go 约5100行实现。
在`runtime.main`函数中，会创建一个M，`sysmon`函数是该M的`mstartfn`函数。M在启动时，`mstart1`函数中会执行一下`mstartfn`函数（如果非nil）然后绑定P、进入调度循环。
而`sysmon`函数是无限循环的，因此这个专门负责执行`sysmon`函数的M会永远停在`sysmon`中，不会进入调度循环。

`sysmon`函数干了这么几件事：
1. 首先检查死锁
2. 进入无限循环
3. 休眠delay时间，delay从20us开始每轮倍增，上限10ms
4. 兜底netpoll：正常情况下M在调度循环会进行netpoll，这里的netpoll是为了防止别处的netpoll没有生效，所以是兜底。如果有恢复可执行的G，将它们放到全局队列。
5. 如果Scavenger就绪，唤醒Scavenger。Scavenger负责周期性地打扫heap中无用的操作系统内存分页。
6. 抢占因系统调用阻塞的P、长时间执行的G，调用`retake`函数。如果发生了抢占，idle（sysmon轮数）归零，这会使循环的休眠时间delay恢复最小的20us。就是说，发生了抢占后，希望sysmon能加快运行速度。
7. 强制GC：检查GC是不是有一段时间（forcegcperiod=2分钟）没有执行过了，如果是，触发强制GC。
8. 回到第3步，无限循环

## retake - 抢占

参考：[6.8 协作与抢占](https://golang.design/under-the-hood/zh-cn/part2runtime/ch06sched/preemption/)

retake负责抢占：1）因系统调用阻塞的P；2）长时间执行的G。

遍历所有P
  - 如果P的状态是`_Prunning` 或 `_Psyscall`，检查它的执行时间是否10ms，如果是，抢占正在执行的G。抢占的方式并非直接打断G的执行，而是这样：
    > 这个抢占不是主动的，而是被动的，通过 preemptone 设置P的 stackguard0 设为 stackPreempt 和 gp.preempt = true ，导致该P中正在执行的G进行下一次函数调用时， 导致栈空间检查失败。进而触发 morestack（汇编代码，位于asm_XXX.s中），然后进行一连串的函数调用，最终会调用 goschedImpl 函数，进行解除P与当前M的关联，让该G进入 _Grunnable 状态，插入全局G列表，等待下次调度。触发的一系列函数如下：morestack() -> newstack() -> gopreempt_m() -> goschedImpl() -> schedule()
  - 如果P的状态是`_Psyscall`，且已经保持这个状态10ms，则用`handoffp`函数将它抢占。但有两个例外情况：
    - P的本地队列为空，这种情况抢占它没有意义，不如等系统调用完成后，让原本的M接管它。
    - 有正在自旋的M，这种情况下P的本地队列中的G会被窃取，不会长时间得不到执行（饿死），因此也不需要抢占。

## handoffp - 交出P

以下三种情况，会用`startm`复用或者创建一个M来接管P：
  - 如果P本地队列非空，或者全局队列非空，
  - 如果GC在黑化阶段，当前P可以进行一些GC任务
  - 当前没有M在自旋，也没有P处于空闲状态

否则，将P放到空闲P列表中。
