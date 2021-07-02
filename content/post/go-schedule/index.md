---
title: "Go Schedule"
description: Go 调度源码分析
date: 2021-07-02T11:55:58+08:00
image: index.jpg
math: 
license: 
hidden: false
comments: true
categories:
    - program
    - go
tags:
    - go
    - source
---
# GMP调度模型

本文Go源码版本为[Go 1.14](https://github.com/lyeka/go/tree/go1.14/src)，示例代码只节选了相关的部分

## Go调度模型发展历史

0. 单线程调度

    - GM模型

1. 多线程调度

    - GM模型

    - 全局共享G队列，锁竞争严重

2. 任务窃取调度

    - GMP模型

    - G需要主动让出CPU资源才能触发调度

    - 饥饿问题

    - STW时间过长

3. 抢占式调度

    1. 基于协作的抢占式调度（Go1.2 - Go1.13)
        - 利用编译器注入函数，G在发送函数调用式会执行注入的函数检查是否需要执行抢占
        - 在GC  STW或者系统监控（sysmon）发行G运行过长时发出抢占请求，让G让出
        - 如果无函数调用，例如for循环将导致无法触发抢占
    2. 基于信号的抢占式调度（Go1.14 - ~)
        1. 程序启动时注册`SIGURG`信号处理函数
        2. GC栈扫描时触发抢占 -> 发送 `SIGURG`信号
        3. 操作系统中断线程，触发之前注册的函数进行一系列调度

4. 非均匀内存访问调度（提案）

    - 拆分全局资源（网络轮询器，计数器等），让各个P就近获取，减少锁竞争，增加数据局部性



## GMP模型

G: Goroutine，用户态线程，Go调度执行的基本单位，Go调度器管理

M: 内核线程，操作系统管理

P: Processor，处理器，提供G运行的上下文

TODO 添加图解模型



## 调度流程

### 调度器初始化

`runtime/proc.go` 里的 `schedinit` 函数注释说明了Go程序的启动顺序

```
// The bootstrap sequence is:
//
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
//
// The new G calls runtime·main.
```

1. 调用osinit
2. 调用schedinit
3. 创建一个G用于调用`runtime.main`
4. 调用`runtime.mstart`



实际上启动函数是在汇编文件中，不同的平台有不同的实现，这里以AMD 64位系统举例

```assembly
// src/runtime/asm_amd64.s
TEXT runtime·rt0_go(SB),NOSPLIT,$0
	...
ok:
	// set the per-goroutine and per-mach "registers"
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX
	MOVQ	CX, g(BX)
	LEAQ	runtime·m0(SB), AX
	
	// m0与g0绑定
	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)

	CLD				// convention is D is always left cleared
	CALL	runtime·check(SB)

	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB) 
	CALL	runtime·osinit(SB) 
	CALL	runtime·schedinit(SB)

	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	PUSHQ	$0			// arg size
	CALL	runtime·newproc(SB)
	POPQ	AX
	POPQ	AX

	// start this M
	CALL	runtime·mstart(SB)
```



#### osinit

```go
// src/runtime/os_linux.go
func osinit() {
	ncpu = getproccount()
	physHugePageSize = getHugePageSize()
	osArchInit()
}
```

osinit在不同的操作系统上有不同的实现，这里列举的是linux平台的实现，主要主要操作是获取CPU个数， HugePage的大小等。



#### schedinit

```go
// src/runtime/proc.go
func schedinit() {
	_g_ := getg()

	sched.maxmcount = 10000
	
	mcommoninit(_g_.m)

	gcinit()

	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}

}
```



schedinit主要是做一些调度器初始化的操作，包括

1. M的初步初始化 -> `mcommoninit(_g_.m)`
2. 垃圾收集初始化 -> `gcinit()`
3. P资源池的初始化，以及M0绑定P -> `procresize(procs)`
4. ...



#### 创建main groutine用于调用`runtime.main`

```go
// src/runtime/proc.go
func main() {
	g := getg()

	if sys.PtrSize == 8 {
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}


	if GOARCH != "wasm" {
		systemstack(func() {
			newm(sysmon, nil)
		})
	}

	if g.m != &m0 {
		throw("runtime.main not on m0")
	}

	gcenable()


	fn := main_main 
	fn()

	exit(0)
}
```

runtime.main实际上是用户定义的main包下main函数的入口，主要包括以下操作

1. 设定栈大小的阈值 -> `maxstacksize`
2. 启动系统监控线程 -> `newm(sysmon, nil)`
3. 启动垃圾回收器 -> `gcenable()`
4. 执行用户main函数 -> `fn := main_main; fn`



#### mstart

```go
// src/runtime/proc.go
func mstart() {
	_g_ := getg()
	mstart1()
}

func mstart1() {
	_g_ := getg()

	if _g_ != _g_.m.g0 {
		throw("bad runtime·mstart")
	}
	
    // 初始化线程，主要注册系统信号的处理函数
	minit()
	if _g_.m == &m0 {
		mstartm0()
	}

	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}

	if _g_.m != &m0 { // 如果不是M0的话需要绑定一个P
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
	schedule()
}


func mstartm0() {
	initsig(false)
}
```



mstart是创建M的入口，Go程序启动时会创建一个主线程m0, 以全局变量在程序生命周期内存在。m0和其它进程中动态创建的M没什么不同，只不过M0是汇编直接创建赋值的。

mstartx这一系列函数主要操作包括

1. 初始化M -> `minit；mstartm0`
2. 执行M的起始函数（如果有的话)
3. 绑定P -> `acquirep(_g_.m.nextp.ptr())`
4. 开始调度 -> `schedule()`



### 循环调度

#### schedule

```go
// src/runtime/proc.go
func schedule() {
	_g_ := getg()

top:	
    // GC STW时休眠M
	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}

	var gp *g
	
	if gp == nil {
		// 1. 为了公平，当全局G队列不为空时，有一定机率从全局G队列获取
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
        // 2. 从P的本地G队列获取G
		gp, inheritTime = runqget(_g_.m.p.ptr())
	}
	if gp == nil {
        // 3. 尝试从别处获取G
		gp, inheritTime = findrunnable() // blocks until work is available
	}
	
    // 执行G上的函数
	execute(gp, inheritTime)
}


```

当一个M绑定了P后，会开始获取G来执行，主要流程如下

1. 获取G，优先级如下：

    1. 为了避免全局G队列的G得不到调度，P每61次调度会优先从全局G队列获取G
    2. 从P的本地G队列获取G
    3. 调用`findrunable`更深层次的获取G，这一步是阻塞的

2. 执行G



#### findrunable

```go
// src/runtime/proc.go
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()

top:
	_p_ := _g_.m.p.ptr()
	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}

	// 1. 还是先尝试从P的本地G队列获取
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}

	// 2. 从全局G队列获取
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}

	// 3. 从netpoller获取
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

    // 4. 从其他P窃取
	procs := uint32(gomaxprocs)
	ranTimer := false
    // 当自旋的M数量大于等非闲置状态下P的数量时，进入Stop流程
    // 主要是为了在减少P很多但是程序并发度很小下CPU的消耗
	if !_g_.m.spinning && 2*atomic.Load(&sched.nmspinning) >= procs-atomic.Load(&sched.npidle) {
		goto stop
	}
    // 切换为自旋状态
	if !_g_.m.spinning {
		_g_.m.spinning = true
		atomic.Xadd(&sched.nmspinning, 1)
	}
	for i := 0; i < 4; i++ {
        // 随机算法得到切换目标P(p2)
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				goto top
			}
			stealRunNextG := i > 2 // first look for ready queues with more than 1 g
			p2 := allp[enum.position()]
			if _p_ == p2 {
				continue
			}
            // 窃取p2一半的G到p的G队列
			if gp := runqsteal(_p_, p2, stealRunNextG); gp != nil {
				return gp, false
			}

			
stop:
	// TODO
}
```

findrunable会按下面优先级获取G：

1. P的本地G队列
2. 全局G队列
3. 网络轮询器
4. 从其他P窃取



### 程序运行中的GMP

GMP模型中，M与P都是由调度器控制的，留给开发者的只有G的控制

#### G

结构字段（部分）

```go
type g struct {
	stack       stack   // 栈的边界[stack.lo, stack.hi)
	stackguard0 uintptr // 用于抢占
	stackguard1 uintptr // offset known to liblink

	_panic       *_panic // panic相关
	_defer       *_defer // defer相关
	m            *m      // 绑定的M
	sched        gobuf   // G调度的上下文
    
    atomicstatus uint32 // G状态
    
	goid         int64  // G ID
	
	// 标记是否可以被抢占
	preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
	preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
	preemptShrink bool // shrink stack at synchronous safe point

	startpc        uintptr         // pc of goroutine function
    
	timer          *timer         // 内部定时器
	selectDone     uint32         // are we participating in a select and did someone win the race?

}
```

G的`sched`字段类型为`gobuf`，其包含了G调度运行的上下文

```go
type gobuf struct {

	sp   uintptr // 栈指针
	pc   uintptr // 程序计数器
	g    guintptr // 关联的G
	ctxt unsafe.Pointer 
	ret  sys.Uintreg // 系统调用返回值
	lr   uintptr
	bp   uintptr // for GOEXPERIMENT=framepointer
}

```

当切换goroutine时，需要切换CPU执行的上下文，主要涉及到寄存器

- SP: 当前G的栈范围
- PC: 下一个要执行指令的地址



##### 创建G

```go
// src/runtime/proc.go
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg()
	pc := getcallerpc()
    // 使用系统栈来调用newproc1来创建G
	systemstack(func() {
		newproc1(fn, argp, siz, gp, pc)
	})
}

func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) {
	_g_ := getg()

	// 检查go的任务函数是否为空
	if fn == nil {
		_g_.m.throwing = -1 // do not dump full stacks
		throw("go of nil func value")
	}
	acquirem() // 通过绑定P到一个本地变量来使其不被抢占
	siz := narg
	siz = (siz + 7) &^ 7

	// 如果函数的参数大小超过限制的话，抛出异常
	// 虽然可以初始化一个更大的G栈，但是没必要，因为参数大小超过限制通常是一个错误
	if siz >= _StackMin-4*sys.RegSize-sys.RegSize {
		throw("newproc: function arguments too large for new goroutine")
	}

	_p_ := _g_.m.p.ptr()
	newg := gfget(_p_) // 从P的gfree队列获取G，gfree里面的G都是已经结束状态，这里可以复用G避免多余的内存分配
	if newg == nil { // 没有可复用的G
		newg = malg(_StackMin) // 申请一个_StackMin（2KB）大小的G
		casgstatus(newg, _Gidle, _Gdead) // 更改G的状态为_Gdead
		allgadd(newg) // 将G放入allg队列避免被GC（因为这时候G的栈还没初始化）
	}
	
    ... // G的一些其它初始化操作
    
	// 初始化G后，更改G的状态为可执行
	casgstatus(newg, _Gdead, _Grunnable)
    
    if _p_.goidcache == _p_.goidcacheend {
		// Sched.goidgen is the last allocated id,
		// this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
		// At startup sched.goidgen=0, so main goroutine receives goid=1.
		_p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
		_p_.goidcache -= _GoidCacheBatch - 1
		_p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
	}
	// 生成G的id
	newg.goid = int64(_p_.goidcache)
	// G id自增操作，在P内保证唯一
	_p_.goidcache++
	
    ...
    
	// 将G放入任务队列
	runqput(_p_, newg, true)

	// 当有空闲的P，且没有自旋的M，且main goroutine已经启动，唤醒某个P来干活
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
		wakep()
	}
	// 当初始化G后，这时候可以释放M来标识可以被抢占了
	releasem(_g_.m)
}
```

当使用`go func() {...}`开启一个goroutine时`runtime`会执行下面流程：

1. 调用`systemstack`在系统栈中由g0调用`newproc1`真正创建G
2. 从P的`gfree`队列中寻找可以复用的G
3. 初始化G的栈以及一些字段
    1. 确定栈范围
    2. 初始化调度上下文`gobuf`
    3. ...
4. 将G排队等待被执行



##### G排队等待被执行

```go
// src/runtime/proc.go
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrand()%2 == 0 {
		next = false
	}

	if next { // 1. 尝试直接放到P的下一个执行G槽位
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		gp = oldnext.ptr()
	}

retry:
	h := atomic.LoadAcq(&_p_.runqhead)
	t := _p_.runqtail
	if t-h < uint32(len(_p_.runq)) { // 2. 本地G还有坑位，G插入队尾
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1)
		return
	}
    // 3. 本地G满了，放置到全局G队列
	if runqputslow(_p_, gp, h, t) {
		return
	}
	goto retry
}
```

G初始化后，会被创建其的P放进一个合适的队列等待执行：

1. 如果next为true的话，尝试直接尝试直接放到P的下一个执行G槽位
    1. 在创建G后执行`runqput`后next都是true；可以让这个新G有一定的几率优先被调度
2. 本地G队列（最大保存256个G)还有坑位的时候，G插入队尾
3. 否则放入全局G队列



##### G执行

```go
// src/runtime/proc.go
func execute(gp *g, inheritTime bool) {
	_g_ := getg()

	// GM绑定
	_g_.m.curg = gp
	gp.m = _g_.m
	// G状态改为运行中
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		_g_.m.p.ptr().schedtick++
	}

	...

	// 正式执行G
	gogo(&gp.sched)
}
```

在循环调度`schedule`中当找到G后，就会调用`execute`执行G：

1. GM绑定
2. 修改G状态为运行中
3. 调用`gogo`正式执行

正式执行G逻辑在汇编代码`runtime·gogo`中



##### G进入系统调用

```go
// src/runtime/proc.go
func entersyscall() {
	reentersyscall(getcallerpc(), getcallersp())
}

func reentersyscall(pc, sp uintptr) {
	_g_ := getg()
    
	...
    
	// 保存G调度器上下文，用于下次被调度时恢复现场
	save(pc, sp)
	_g_.syscallsp = sp
	_g_.syscallpc = pc
	// 修改G状态为系统调用中
	casgstatus(_g_, _Grunning, _Gsyscall)
	
    ...
	
    // 将P的M指针置零
	pp := _g_.m.p.ptr()
	pp.m = 0
	// M还是会把旧P的指针保留下（oldp), 用来当系统调用唤醒后优先绑定这个P
	_g_.m.oldp.set(pp)
	_g_.m.p = 0
	// 修改P的状态为系统调用，以便于其他M绑定这个P来执行工作
	atomic.Store(&pp.status, _Psyscall)
	
    ...
}

func entersyscallblock() {
	_g_ := getg()
    
    ... 
	pc := getcallerpc()
	sp := getcallersp()
	save(pc, sp)
    
    ...
    
	casgstatus(_g_, _Grunning, _Gsyscall)
	
	// entersyscallblock_handoff会执行释放P的操作
	systemstack(entersyscallblock_handoff)

	...
}
```

当执行G的过程中陷入了系统调用，汇编会调用`entersyscall`或者`entersyscallblock`来执行调度操作：

1. 保存G调度器上下文，用于下次被调度时恢复现场

2. MP（半）解绑，P此时可以被其他M绑定以执行其他任务



`reentersyscall`与`entersyscallblock`的区别在于，在知道这次系统调用会阻塞的时候会调用`entersyscallblock`，这样的会直接释放掉P给别的M来抢占；而`reentersyscall`不会直接释放掉P，只是将修改P的状态和M指针置零，系统线程会`sysmon`会监控这次系统调用，如果调用时间太长了，才完全释放P被抢占



##### G退出系统调用

```go
// src/runtime/proc.go
func exitsyscall() {
	_g_ := getg()

	// G和M不改变绑定关系，寻找空闲的P绑定
	// GM不解绑的话相对高效一些, 所以优先尝试这种方式
	oldp := _g_.m.oldp.ptr()
	_g_.m.oldp = 0
	if exitsyscallfast(oldp) {
		...
        
         // 绑定了P, 在恢复执行前修改G状态为执行中
		casgstatus(_g_, _Gsyscall, _Grunning)

		...

		if sched.disable.user && !schedEnabled(_g_) {
			// Scheduling of this goroutine is disabled.
			Gosched()
		}

		return
	}

	// 解绑GM，尝试获取P
	mcall(exitsyscall0)
	
    ...
}

func exitsyscallfast(oldp *p) bool {
	_g_ := getg()

    ...

	// 尝试去抢占之前M之前绑定的P（oldp)
	// 如果之前的P处于空闲状态(_Psyscall)
	if oldp != nil && oldp.status == _Psyscall && atomic.Cas(&oldp.status, _Psyscall, _Pidle) {
		// There's a cpu for us, so we can run.
		// 可以的话绑定这个P
		wirep(oldp)
		exitsyscallfast_reacquired()
		return true
	}

	//  获取全局其余空闲的P
	if sched.pidle != 0 {
		var ok bool
		systemstack(func() {
			ok = exitsyscallfast_pidle()
			if ok && trace.enabled {
				if oldp != nil {
					// Wait till traceGoSysBlock event is emitted.
					// This ensures consistency of the trace (the goroutine is started after it is blocked).
					for oldp.syscalltick == _g_.m.syscalltick {
						osyield()
					}
				}
				traceGoSysExit(0)
			}
		})
		if ok {
			return true
		}
	}
	return false
}

func exitsyscall0(gp *g) {
	_g_ := getg()

	// 将G的状态改为可运行（_Grunnable）
	casgstatus(gp, _Gsyscall, _Grunnable)
	// 解绑GM关系
	dropg()
	lock(&sched.lock)
	var _p_ *p
	if schedEnabled(_g_) {
		// 还是优先去看有没有空闲的P来绑定
		_p_ = pidleget()
	}
	if _p_ == nil {
		// 还没没有空闲的P, 将G放进全局G队列
		globrunqput(gp)
	} else if atomic.Load(&sched.sysmonwait) != 0 {
		atomic.Store(&sched.sysmonwait, 0)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)
	if _p_ != nil {
		// 找到空闲P绑定的话，执行G
		acquirep(_p_)
		execute(gp, false) // Never returns.
	}
	if _g_.m.lockedg != 0 {
		// Wait until another thread schedules gp and so m again.
		stoplockedm()
		execute(gp, false) // Never returns.
	}
	// 没有P绑定的话，停止这个M->阻塞，
	// 直到有新的任务->唤醒
	stopm()
    // 唤醒后开始循环调度
	schedule() // Never returns.
}
```

当G从系统调用阻塞中唤醒，汇编调用`exitsyscall`来使其准备被调度运行，M此时会尝试去绑定一个P来执行，有两个路径执行

1. 快速路径`exitsyscallfast`
    1. 尝试去绑定陷入系统调用之前绑定的P
    2. 无法绑定旧P的话，尝试绑定全局空闲的P
2. 慢路径`exitsyscall0`
    1. GM解绑，
    2. 还是尝试去寻找P来绑定，如果找到了， MP绑定，执行G（此时GM又会重新绑定）
    3. 找不到P的话，将G放入全局G队列
    4. 停止这个M直到有新的任务

##### G退出

```go
// src/runtime/proc.go
func goexit1() {
	...
	mcall(goexit0)
}

// goexit continuation on g0.
func goexit0(gp *g) {
	_g_ := getg()

	// G清理工作
	casgstatus(gp, _Grunning, _Gdead)
	...
	gp.m = nil
	...

	// GM解绑
	dropg()

	...

	// 将G放入gfree队列待复用
	gfput(_g_.m.p.ptr(), gp)
	
    ...
    
	// 循环调度
	schedule()
}
```

当G的任务函数逻辑执行完毕后，会调用`goexit1`执行清理退出工作：

1. 调用`mcall`切换到g0来调用`goexit0`执行真正的退出工作
2. 修改G的状态为结束
3. GM解绑
4. 开始新一轮的循环调度



#### M

##### 创建M

```go
// src/runtime/proc.go
func newm(fn func(), _p_ *p) {
	// 创建M
	mp := allocm(_p_, fn)
    // 设置P
	mp.nextp.set(_p_)
	mp.sigmask = initSigmask
	... // 
	newm1(mp)
}

func newm1(mp *m) {
	...
	execLock.rlock() // Prevent process clone.
	newosproc(mp)
	execLock.runlock()
}
```

创建M的起始函数是`newm`，这里做了一些字段的初始化，真正的内核线程创建在`newosproc`中实现

```go
// src/runtime/os_linux.go
func newosproc(mp *m) {
	stk := unsafe.Pointer(mp.g0.stack.hi)
	...
	var oset sigset
	sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
	// clone通过系统调用来真正创建系统线程
	// 创建系统线程后，执行mstart
	ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
	sigprocmask(_SIG_SETMASK, &oset, nil)
	...
}
```

当`clone`创建内核线程w安毕后，M就会执行`mstart`， 关于`mstart`, 前面调度器初始化部分已经讲到，这里不再赘述，总之，在创建完M之后，每个M就会进入循环调度阶段。

##### 什么时候创建M

TODO





### 窃取与抢占

TODO

### 系统线程

TODO



## 一些术语

### systemstack

在系统栈上执行fn，在g0栈或者信号处理栈上调用

### mcall

切换到g0栈来执行任务函数fn，必须在用户逻辑的g栈来调用。

与systemstack不同的是，fn必须永不返回，通常会以循环调度`schedule`结束

### g0

每个M拥有两个G，分别为承载用户逻辑的g以及调度逻辑的g0，在运行一些runtime调度相关的逻辑时，会切换到g0来执行



## ref

- [Scalable Go Scheduler Design Doc](https://golang.org/s/go11sched.)
- [work-stealing scheduler](http://supertech.csail.mit.edu/papers/steal.pdf)
- [Go语言原本- 6.3 MPG 模型与并发调度单元](https://golang.design/under-the-hood/zh-cn/part2runtime/ch06sched/mpg/)
- [Go语言原本 - 6.4 调度循环](https://golang.design/under-the-hood/zh-cn/part2runtime/ch06sched/schedule/#641-)
- [深入golang runtime的调度](https://zboya.github.io/post/go_scheduler/)
- [ Go设计与实现 - 6.5 调度器 ](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#65-调度器)
- [详解 Go 程序的启动流程，你知道 g0，m0 是什么吗？](https://developer.51cto.com/art/202104/656994.htm)
- [mcall systemstack等汇编函数](https://studygolang.com/articles/28553)
- [参与运行时的系统调用](https://golang.design/under-the-hood/zh-cn/part2runtime/ch10abi/syscallrt/)

