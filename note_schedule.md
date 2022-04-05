[TOC]

# schedule函数调度实现

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
			// 上限是1，不会批量转移G到P的本地队列，只返回一个G（如果有）
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


# findrunnable - 找到可用的G

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
	// 检查是否有恢复可执行的G，如果有，就把G拿来执行。
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
4. 检查netpoll：检查是否有恢复可执行的G，如果有，就把G拿来执行
5. 工作窃取：如果当前M正在自旋，或者没在自旋但所有自旋M数量不到工作中P数量的一半，则可以进入自旋，并进行工作窃取。
6. GC任务：如果目前GC在特定阶段（黑化），则获取一个执行GC的G
7. 再一次尝试从全局队列获取G，如果还是获取不到，释放P，将P改为休眠状态。
8. 如果M处于自旋状态，则再次检查是否以下特定情况
    - 有没有空闲，但本地队列非空的P。如果有，绑定P，并回到第1步尝试获取G。
    - 有没有GC任务可运行的P，如果有，绑定P并返回一个执行GC的G。
  如果不是这两种特殊情况，则不再自旋。
  另外，在这里还会检查所有P中，最快到期的timer，还有多久到期，赋值给pollUntil变量。
9. 以pollUntil变量为上限，阻塞检查netpoll。如果从netpoll获取到有G恢复可执行，且能够获取到一个空闲的P，则绑定P并获取一个G。
9. 执行到这里，表示所有尝试都没有找到G，则M进入休眠，等待被唤醒。M被唤醒时，会立刻绑定一个P，并回到第1步重新开始找G。

注意：`findrunnable`函数是被`schedule`函数调用的，在找到可执行的G之前，M要么自旋要么休眠，都没有从`findrunnable`函数返回。找到可执行的G之后，才会将G返回给`schedule`函数，由`schedule`函数来`execute`执行。
