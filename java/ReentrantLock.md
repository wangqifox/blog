---
title: ReentrantLock
date: 2017/09/29 09:33:25
---

lock
	acquire(1)
		tryAcquire	acquireQueued	addWaiter
lockInterruptibly
	acquireInterruptibly(1)
		tryAcquire	doAcquireInterruptibly
unlock
	release(1)
		tryRelease
<!--more-->
## Semaphore

acquire
	acquireSharedInterruptibly(1)
		tryAcquireShared	doAcquireSharedInterruptibly
acquireUninterruptibly
	acquireShared(1)
		tryAcquireShared	doAcquireShared
release
	releaseShared(1)
		tryReleaseShared	doReleaseShared

## CountDownLatch

await
	acquireSharedInterruptibly(1)
		tryAcquireShared	doAcquireSharedInterruptibly
countDown
	releaseShared(1)
		tryReleaseShared	doReleaseShared

## ReentrantRead

ReadLock.lock
	acquireShared
		tryAcquireShared	doAcquireShared

ReadLock.unlock
	releaseShared
		tryReleaseShared	doReleaseShared

WriteLock.lock
	acquire
		tryAcquire	acquireQueued	addWaiter

WriteLock.unlock
	release
		tryRelease

用到的方法：

acquire
	独占模式获取对象，忽略中断

acquireInterruptibly
	独占模式获取对象，响应中断

acquireShared
	共享模式获取对象，忽略中断

acquireSharedInterruptibly
	共享模式获取对象，响应中断

release
	独占模式释放对象

releaseShared
	共享模式释放对象

重写的方法：

tryAcquire
tryAcquireShared
tryRelease
tryReleaseShared


