[TOC]

# 总览

Golang程序入口：[asm_amd64.s](src\runtime\asm_amd64.s)，`TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0`函数。

1. 创建g0、m0，相互绑定，asm_amd64.s 266行左右
2. `runtime·osinit` 操作系统相关的初始化
3. `runtime·schedinit` 调度系统初始化
4. `runtime·mainPC`取出`runtime.main`函数的值（PC：程序计数器），`runtime·newproc`创建main goroutine，目标函数是`runtime.main`
5. `runtime·mstart` 启动M

## osinit 操作系统相关的初始化

这个方法对每种操作系统都有一个特定实现，因此是“操作系统相关的”。在这个方法中执行一些和操作系统相关的初始化操作，比如在os_linux.go中就定义了linux操作系统的初始化操作：将cpu数量计算出来，并保存在一个叫`ncpu`的变量。

## schedinit 调度系统初始化

与`osinit`方法相比，`schedinit`方法只有一个实现，可见它是操作系统无关的、通用的。

`schedinit`方法中调用了相当多`xxxinit`方法，表示各种初始化。这里先不一个个说。

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

`newproc`函数在proc.go中实现，
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
- 用传入的函数值fn创建一个goroutine `newg`
- 把`newg`放到当前G的P的G队列里

当前G，在这种情况下，就是g0，它的P就是`runtime.schedinit`创建的所有P中的一个。此时m0和g0绑定，它们和同一个P绑定，`newg`也被放到这个P的G队列中。

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

## schedule函数调度实现

```java
func schedule() {
	_g_ := getg()

	...

top:
	// 如果当前GC需要停止整个世界（STW), 则调用gcstopm休眠当前的M
	if sched.gcwaiting != 0 {
		// 为了STW，停止当前的M
		gcstopm()
		// STW结束后回到 top
		goto top
	}

	// 检查有无需要执行的timer
	checkTimers(pp, 0)

	var gp *g
	var inheritTime bool

	...
	
	if gp == nil {
		// 每隔61次调度，尝试从全局队列中获取G
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		// 从p的本地队列中获取
		gp, inheritTime = runqget(_g_.m.p.ptr())
	}
	if gp == nil {
		// 想尽办法找到可运行的G，比如从p本地队列、全局队列获取。
		// 如果暂时找不到，就在这个方法里阻塞。
		gp, inheritTime = findrunnable() // blocks until work is available
	}

	...

	// println("execute goroutine", gp.goid)
	// 找到了g，那就执行g上的任务函数
	execute(gp, inheritTime)
}
```

1. 如果当前GC需要停止整个世界（STW), 则调用gcstopm休眠当前的M。
2. 检查一下有没有timer能够执行了，timer是time.Timer和time.Ticker的底层实现。
3. 每调度61次，从全局队列获取1个G，避免全局队列中的G被饿死。
4. 如果上一步没有找到G，从P的本地队列获取G。
5. 如果上一步没有找到G，调用 findrunnable 找G，找不到的话就将M休眠，等待唤醒。
6. 调用 execute 执行G

因为前面`runtime.mainPC`、`runtime.newproc`已经创建好main goroutine并添加到P的本地队列中，所以在`schedule`第4步能够找到main goroutine并开始执行它。

## main goroutine开始执行

main goroutine的目标函数是`runtime.main`，在proc.go 约145行实现。

1. 创建一个M专门负责执行sysmon（系统监控）函数，这个M会在sysmon中无限循环，不会（像普通M一样）进入调度循环。
2. `doInit(&runtime_inittask)`执行runtime包里的init函数
3. `gcenable()`创建2个负责GC的goroutine
4. `doInit(&main_inittask)`执行main包里的init函数，也就是开发者自己写的init函数。
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

## findrunnable - 找到可用的G

```go
// 找到一个可以运行的G，不找到就让M休眠，然后等待唤醒，直到找到一个G返回
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()

top:
	...

	// 检查有无需要执行的timer
	now, pollUntil, _ := checkTimers(_p_, 0)

	...

	// 从P本地队列获取G，如果获取到就返回
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}

	// 从全局队列获取G，如果获取到就返回
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}

	// 检查netpoll（网络轮询）
	// 检查是否有因网络就绪而变为可执行的G，如果有，就把G拿来执行。
	// 条件：网络轮询已初始化 && 存在等待网络的G（waiter） && 当前没有（别的线程）在进行网络轮询检查。
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}

	// 工作窃取
	// 如果当前M正在自旋，或者没在自旋但所有自旋M数量不到工作中P数量的一半，
	// 则可以进入自旋，并进行工作窃取。
	procs := uint32(gomaxprocs)
	nBusyP := procs - atomic.Load(&sched.npidle)
	if _g_.m.spinning || 2*atomic.Load(&sched.nmspinning) < nBusyP {
		if !_g_.m.spinning {
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)
		}

		gp, inheritTime, tnow, w, newWork := stealWork(now)
		now = tnow
		if gp != nil {
			// Successfully stole.
			return gp, inheritTime
		}
		if newWork {
			// There may be new timer or GC work; restart to
			// discover.
			goto top
		}
		if w != 0 && (pollUntil == 0 || w < pollUntil) {
			// Earlier timer to wait for.
			pollUntil = w
		}
	}

	// 在GC的黑化阶段，获取一个执行GC的G
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) {
		node := (*gcBgMarkWorkerNode)(gcBgMarkWorkerPool.pop())
		if node != nil {
			_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
			gp := node.gp.ptr()
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}


	// 保存现场
	allpSnapshot := allp
	idlepMaskSnapshot := idlepMask
	timerpMaskSnapshot := timerpMask

	// return P and block
	lock(&sched.lock)
	if sched.gcwaiting != 0 || _p_.runSafePointFn != 0 {
		unlock(&sched.lock)
		goto top
	}
	// 再一次尝试从全局队列获取G
	if sched.runqsize != 0 {
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		return gp, false
	}
	// 取消P、M绑定
	if releasep() != _p_ {
		throw("findrunnable: wrong p")
	}
	// 将P设为休眠
	pidleput(_p_)
	unlock(&sched.lock)

	// 先把M自旋状态设置为“非自旋”，然后重试和上面类似的获取G步骤
	wasSpinning := _g_.m.spinning
	if _g_.m.spinning {
		_g_.m.spinning = false
		if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
			throw("findrunnable: negative nmspinning")
		}

		// 再次检查 有没有空闲，但本地队列非空的P
		// 如果有，绑定P，重新开始自旋，并回到开头尝试获取G
		// Check all runqueues once again.
		_p_ = checkRunqsNoP(allpSnapshot, idlepMaskSnapshot)
		if _p_ != nil {
			acquirep(_p_)
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)
			goto top
		}

		// 再次检查 有没有GC任务可运行的P，如果有，绑定P并执行GC任务
		// Check for idle-priority GC work again.
		_p_, gp = checkIdleGCNoP()
		if _p_ != nil {
			acquirep(_p_)
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)

			// Run the idle worker.
			_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}

		pollUntil = checkTimersNoP(allpSnapshot, timerpMaskSnapshot, pollUntil)
	}

	// 再次检查netpoll
	...

	// 到最后都没有找到G，则M进入休眠
	stopm()
	// M被唤醒后，重新回到开头寻找一个可执行的G
	goto top
}
```

1. 检查一下有没有timer能够执行了，这一点和schedule方法一样
2. 从本地队列获取G
3. 从全局队列获取G
4. 检查netpoll：检查是否有因网络就绪而变为可执行的G，如果有，就把G拿来执行
5. 工作窃取：如果当前M正在自旋，或者没在自旋但所有自旋M数量不到工作中P数量的一半，则可以进入自旋，并进行工作窃取。
6. GC任务：如果目前GC在特定阶段（黑化），则获取一个执行GC的G
7. 再一次尝试从全局队列获取G，如果还是获取不到，释放P，将P改为休眠状态。
8. 如果M处于自旋状态，则再次检查是否以下特定情况
  - 有没有空闲，但本地队列非空的P。如果有，绑定P，并回到第1步尝试获取G。
  - 有没有GC任务可运行的P，如果有，绑定P并返回一个执行GC的G。
  如果不是这两种特殊情况，则不再自旋。
  另外，在这里还会检查所有P中，最快到期的timer，还有多久到期，赋值给pollUntil变量。
9. 以pollUntil变量为上限，阻塞检查netpoll。如果从netpoll获取到有G恢复可用了，且能够获取到一个空闲的P，则绑定P并获取一个G。
9. 执行到这里，表示所有尝试都没有找到G，则M进入休眠，等待被唤醒。M被唤醒时，会立刻绑定一个P，并回到第1步重新开始找G。

注意：`findrunnable`函数是被`schedule`函数调用的，在找到可执行的G之前，M要么自旋要么休眠，都没有从`findrunnable`函数返回。找到可执行的G之后，才会将G返回给`schedule`函数，由`schedule`函数来`execute`执行。

## newproc - 创建新的G

## sysmon - 系统监控