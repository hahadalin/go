[TOC]

# 总览

Golang程序入口：asm_amd64.s，`TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0`函数。

1. 创建g0、m0，相互绑定，asm_amd64.s 266行左右
2. `runtime·osinit` 操作系统相关的初始化
3. `runtime·schedinit` 调度系统初始化
4. `runtime·mainPC`取出`runtime.main`函数的值（PC：程序计数器），`runtime·newproc`创建main goroutine，目标函数是`runtime.main`
5. `runtime·mstart` 启动M

## osinit 操作系统相关的初始化

这个方法对每种操作系统都有一个特定实现，因此是“操作系统相关的”。在这个方法中执行一些和操作系统相关的初始化操作，比如在os_linux.go中就定义了linux操作系统的初始化操作：将cpu数量计算出来，并保存在一个叫`ncpu`的变量。

## schedinit 调度系统初始化

与`osinit`方法相比，`schedinit`函数只有一个实现，可见它是操作系统无关的、通用的。

`schedinit`方法中调用了相当多`xxxinit`函数，表示各种初始化。这里先不一个个说。

在asm_amd64.s约685行，`sched.maxmcount = 10000`表示M的上限是10000。在Golang程序中，并发程度的控制主要由P的数量`GOMAXPROCS`控制，M的数量上限10000一般达不到，是个摆设。

在proc.go约717行，有以下代码
```go
procs := ncpu
if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
    procs = n
}
if procresize(procs) != nil {
    throw("unknown runnable goroutine during bootstrap")
}
```
可以看到，`procs`是P的数量，默认是ncpu（在`osinit`函数中初始化该变量为cpu数量）。如果环境变量中定义了`GOMAXPROCS`，则`procs`取该环境变量的值。确定了`procs`后，用`procresize(procs)`初始化创建所有P（程序启动后也可以修改P的数量，实际上还是通过这个函数）。

在`schedinit`函数的头部注释有说明：
```go
// The bootstrap sequence is:
//
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
//
// The new G calls runtime·main.
```
这和[总览](#总览)中说的顺序一样。

## 创建main goroutine

在asm_amd64.s约355行，先后调了`runtime.mainPC`、`runtime.newproc`两个函数。
`mainPC`没有实现，但注释说
```go
// mainPC is a function value for runtime.main, to be passed to newproc.
// The reference to runtime.main is made via ABIInternal, since the
// actual function (not the ABI0 wrapper) is needed by newproc.
```
即，`mainPC`代表了`runtime.main`的函数值。

`newproc`函数见[newproc---创建新的g](#newproc---创建新的g)，它会创建以`runtime.main`为目标函数的goroutine，也就是main goroutine。

总结：
`runtime.mainPC`拿到了`runtime.main`函数的函数值，`runtime.newproc`为这个函数值创建了一个goroutine，就是所谓**main goroutine**，并把它放到g0和m0绑定的P的G队列中。

## runtime·mstart 启动m0

`mstart`是在asm_amd64.s中实现的汇编函数，它直接调用`runtime.mstart0()`函数，`mstart0()`又直接调`mstart1()`。

```go
func mstart1() {
	// 任何M在启动时，首先获取到的G都是它的g0
	_g_ := getg()

	// 理论上这个校验不会失败
	if _g_ != _g_.m.g0 {
		throw("bad runtime·mstart")
	}

    // 省略一些代码

	// 初始化m
	minit()

	// 对m0的特殊处理：已知_g_是g0，如果g0的M是m0，执行mstartm0()
	if _g_.m == &m0 {
		mstartm0()
	}

	// 如果有m的起始任务函数，则执行，比如 sysmon 函数
	// 对于m0来说，是没有 mstartfn 的
	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}

	// 如果M不是m0，在这里绑定一个P。
	// m0因为schedinit时就已经绑定P了，这里就不用再绑定P。
	if _g_.m != &m0 {
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
	// 进入调度，不会再返回
	schedule()
}
```

可以看到不管是`mstart0`还是`mstart1`都是在做一些准备工作，还没有开始调度goroutine，就连main goroutine都没有开始执行。事实如此，goroutine的调度是`schedule`函数负责的。

`schedule`函数的详细分析见[schedule函数调度实现](note_schedule.md)，这里只需要知道，`schedule`会从P的本地队列取出main goroutine并开始执行它。

## main goroutine开始执行

main goroutine的目标函数是`runtime.main`，在proc.go 约145行实现。

1. 将 `mainStarted` 标识为true，表示main goroutine已经启动，也就是说m0、调度系统的初始化已经完成了，使得（newproc）新建goroutine后尝试唤醒、新建M来执行G。
1. 创建一个M专门负责执行sysmon（系统监控）函数，这个M会在sysmon中无限循环，不会（像普通M一样）进入调度循环。
2. `doInit(&runtime_inittask)`执行runtime包里的init函数
3. `gcenable()`创建2个负责GC的goroutine
4. `doInit(&main_inittask)`执行main包里的init函数，也就是开发者自己写的init函数。
  main包里一般会引用其他包，其他包的init函数会先于main包的init函数执行。main包的init函数会最后被执行，但一定是在这一步。
5. 执行`main_main`函数，它是由编译器链接到`main.main`函数的，就等同于执行`main.main`。
6. 退出进程，除了`exit(0)`正常退出，还写了个有错的语句来确保一定能够退出。

# 细节

## getg - 获取当前G

`gets`函数在stubs.go中定义，但没有实现
```go
// getg returns the pointer to the current g.
// The compiler rewrites calls to this function into instructions
// that fetch the g directly (from TLS or from the dedicated register).
func getg() *g
```
就是说：`getg`返回“当前G”，利用了内核线程的TLS（线程本地存储）。

TLS在[这篇博文](https://zboya.github.io/post/go_scheduler/)中有说明:
>TLS全称是Thread Local Storage，代表每个线程中的本地数据。写入TLS中的数据不会干扰到其余线程中的值。Go的协程实现非常依赖于TLS机制，会用于获取系统线程中当前的G和G所属于的M实例。 Go操作TLS会使用系统原生的接口，以Linux X64为例，go在新建M时候会调用 arch_prctl 这个syscall来设置FS寄存器的值为M.tls的地址，运行中每个M的FS寄存器都会指向它们对应的M实例的tls， linux内核调度线程时FS寄存器会跟着线程一起切换，这样go代码只需要访问FS寄存器就可以获取到线程本地的数据。

就是说：M会把自己的tls字段（指针数组）放到内核线程的TLS中，因为M代表了内核线程，它们的切换是同步的，因此内核线程TLS中始终可以取到对应M的tls字段，从而不难取到对应M的实例m和它当前的G实例g。

## newproc - 创建新的G

`newproc`函数在proc.go 约4100行实现，
```go
// Create a new g running fn.
// Put it on the queue of g's waiting to run.
// The compiler turns a go statement into a call to this.
func newproc(fn *funcval) {
	gp := getg()
	pc := getcallerpc()
	systemstack(func() {
		newg := newproc1(fn, gp, pc)

		_p_ := getg().m.p.ptr()
		runqput(_p_, newg, true)

		if mainStarted {
			wakep()
		}
	})
}
```
根据注释、代码可以看出，它干的事有：
- 用`getg()`取得当前的G
- 调用`newproc1`函数，为传入的函数值fn创建一个goroutine `newg`
- 把`newg`放到当前G的M的P的G队列里
- 如果 `mainStarted` （main goroutine已经启动），`wakep()`复用或者创建一个M并让它自旋。

更多底层实现位于`newproc1`函数中：
- 先尝试从gfree list获取一个G实例，得以复用G
- 没有可复用的G，新建一个G，栈的大小是2K
- 计算栈的很多内部数据，比如栈的大小、边界、pc等等
- 保存G的gopc（用go创建goroutine的语句的PC）、ancestors（祖先G信息）、startpc（目标函数）
- 为goroutine生成唯一的goid

## systemstack - 以系统栈执行函数

在很多地方都有这样的调用：

```go
systemstack(func() {
	// do something
})
```

`systemstack`函数在stubs.go中定义，但并没有实现，其实现是平台相关的，因此不同的平台有不同的汇编实现，比如amd64的实现就位于asm_amd64.s汇编文件中。

在`systemstack`函数的定义上有这样的注释：
>systemstack 在系统堆栈上运行 fn。
>如果从 per-OS-thread (g0) 堆栈调用 systemstack，或者从信号处理 (gsignal) 堆栈调用 systemstack，则 systemstack 直接调用 fn 并返回。
>否则，系统堆栈将从普通 goroutine 的有限堆栈中调用。 在这种情况下，systemstack 切换到 per-OS-thread 堆栈，调用 fn，然后切换回来。

可见，`systemstack`函数的作用就是：在系统堆栈上运行一个指定的函数fn。
所谓系统堆栈，简单理解就是g0所代表的堆栈。在系统堆栈上执行的任务是不被抢占、不被GC扫描的，因此一些比较底层的调用就要求只能在系统堆栈上执行。

具体来说，虽然`systemstack`函数的实现是平台相关的，但基本逻辑是一样的：
- 如果调用`systemstack`函数的当前G就是g0或者gsignal，则fn可以直接调用并返回。
- 否则
    - 保存当前G的现场到g.sched数据结构。
	- 切换到g0堆栈
	- 执行fn
	- 恢复原本的G的现场
	- 返回

和`systemstack`函数类似的还有`mcall`函数、`asmcgocall`函数，它们的作用是类似的：在系统堆栈上调用指定的函数fn，只不过入参不同，而且它们都是平台相关的，都用汇编实现。
