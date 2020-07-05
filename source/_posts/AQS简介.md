---
title: AQS简介
date: 2019-07-23 19:27:42
tags:
    - Java
    - 并发
    - 技术
---

# AQS_5W1H
## 参考资料
`圈圈_Master` - Java技术之AQS详解 - https://www.jianshu.com/p/da9d051dcc3d
`Doug Lea, etc`-《Java并发编程实战》

<!--more-->

## What
`AbstractQueuedSynchronizer`的简称。

## When
JDK5

## Where
America

## Who
Doug Lea（Java巨佬）

## Why
实现阻塞锁和一系列依赖FIFO等待队列的同步器的框架

## How
* 基本是围绕着state（资源）的原子操作
### state
维护了`volatie int state`（不保证操作的原子性，但是保证操作的可见性）--可以延伸
#### 访问方式_均为原子操作
* `getState()`
* `setState()`
* `Unsafe`的`compareAndSwapInt()`方法

### 资源共享方式
* Exclusive（独占，只有一个线程能执行，如ReentrantLock）
* Share（共享，多个线程可同时执行，如Semaphore/CountDownLatch）

### 维护的实现
自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。
#### 实现的主要方法
* `isHeldExclusively()`：该线程是否正在独占资源。只有用到condition才需要去实现它。
* `tryAcquire(int)`：独占方式。尝试获取资源，成功则返回true，失败则返回false。
* `tryRelease(int)`：独占方式。尝试释放资源，成功则返回true，失败则返回false。
* `tryAcquireShared(int)`：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
* `tryReleaseShared(int)`：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

#### 主要方法的源码实现-独占资源获取
##### acquire(int)
* 以独占方式获取资源
* 如果获取到了资源，线程直接返回
* 若没有，则进入等待队列，直到获取资源为止
* 用于独占模式
    ```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    ```
* 代码流程如下：
    1. `tryAcquire()`尝试直接去获取资源，如果成功则直接返回；
    2. `addWaiter()`将该线程加入等待队列的尾部，并标记为独占模式；
    3. `acquireQueued()`使线程在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
    4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断`selfInterrupt()`，将中断补上。

##### tryAcquire(int)
* tryAcquire尝试以独占的方式获取资源
* 如果获取成功，则直接返回true，否则直接返回false。
* 该方法可以用于实现Lock中的`tryLock()`方法。
* 该方法的默认实现是抛出UnsupportedOperationException，具体实现由自定义的扩展了AQS的同步类来实现。
* 这里之所以没有定义成abstract，是因为独占模式下只用实现`tryAcquire-tryRelease`，而共享模式下只用实现`tryAcquireShared-tryReleaseShared`。如果都定义成abstract，那么每个模式也要去实现另一模式下的接口。
    ```java
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
    ```

##### addWaiter(Node)
* 将当前线程根据不同的模式（`Node.EXCLUSIVE`互斥模式、`Node.SHARED`共享模式）加入到等待队列的队尾，并返回当前线程所在的结点。
* 如果队列不为空，则以通过`compareAndSetTail`方法以CAS的方式将当前线程节点加入到等待队列的末尾。
* 否则，通过enq(node)方法初始化一个等待队列，并返回当前节点。(没有则新建队列)
    ```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
    ```

##### enq(node)
* enq(node)用于将当前节点插入等待队列
* 如果队列为空，则初始化当前队列。
* 整个过程以CAS自旋的方式进行，直到成功加入队尾为止。
    ```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
    ```

##### acquireQueued(Node, int)
* `acquireQueued()`用于队列中的线程自旋地以独占且不可中断的方式获取同步状态（acquire），直到拿到`锁`之后再返回。
* 该方法的实现分成两部分：如果当前节点已经成为头结点，尝试获取锁（tryAcquire）成功，然后返回；
* 否则检查当前节点是否应该被park，然后将该线程park并且检查当前线程是否被可以被中断。
    ```java
    final boolean acquireQueued(final Node node, int arg) {
        //标记是否成功拿到资源，默认false
        boolean failed = true;
        try {
            boolean interrupted = false;//标记等待过程中是否被中断过
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    ```

##### shouldParkAfterFailedAcquire(Node, Node)
* shouldParkAfterFailedAcquire方法通过对当前节点的前一个节点的状态进行判断，对当前节点做出不同的操作
* 至于每个Node的状态表示，可以参考接口文档。
    ```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    ```

##### parkAndCheckInterrupt()
* 该方法让线程去休息，真正进入等待状态。
* park()会让当前线程进入waiting状态。
* 在此状态下，有两种途径可以唤醒该线程：1）被unpark()；2）被interrupt()。
* 需要注意的是，Thread.interrupted()会清除当前线程的中断标记位。
    ```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
    ```

##### 小结
我们再回到`acquireQueued()`，总结下该函数的具体流程：
1. 结点进入队尾后，检查状态，找到安全休息点；
2. 调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；
3. 被唤醒后，看自己是不是有资格能拿到号。
4. 如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；
5. 如果没拿到，继续流程1。

最后，总结一下acquire()的流程：
1. 调用自定义同步器的tryAcquire()尝试直接去获取资源，如果成功则直接返回；
2. 没成功，则addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3. acquireQueued()使线程在等待队列中休息，有机会时（轮到自己，会被unpark()）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

#### 主要方法的源码实现-独占资源释放
##### 从release(int)开始
* `release(int)`方法是独占模式下线程释放共享资源的顶层入口。
* 它会释放指定量的资源，如果彻底释放了（即state=0）
* 它会唤醒等待队列里的其他线程来获取资源。
* 这也正是unlock()的语义，当然不仅仅只限于unlock()。
    ```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    ```
    ```java
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
    ```
    ```java
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
    ```
##### 小结
* 与acquire()方法中的tryAcquire()类似，tryRelease()方法也是需要独占模式的自定义同步器去实现的。
* 正常来说，tryRelease()都会成功的，因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可(state-=arg)，也不需要考虑线程安全的问题。
* 但要注意它的返回值，上面已经提到了，release()是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回true，否则返回false。
* unparkSuccessor(Node)方法用于唤醒等待队列中下一个线程。这里要注意的是，下一个线程并不一定是当前节点的next节点，而是下一个可以用来唤醒的线程，如果这个节点存在，调用unpark()方法唤醒。
* 总之，release()是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释* 了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。

#### 主要方法的源码实现-共享资源获取
* acquireShared(int)方法是共享模式下线程获取共享资源的顶层入口。
* 它会获取指定量的资源，获取成功则直接返回，获取失败则进入等待队列，直到获取到资源为止，整个过程忽略中断。
##### acquireShared(int) 
    ```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
    ```

##### doAcquireShared(int)
* acquireShared(int)方法是共享模式下线程获取共享资源的顶层入口。
* 它会获取指定量的资源，获取成功则直接返回，获取失败则进入等待队列，直到获取到资源为止，整个过程忽略中断。
    ```java
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    ```
##### setHeadAndPropagate(Node node, int propagate)
* 跟独占模式比，还有一点需要注意的是，这里只有线程是head.next时（“老二”），才会去尝试获取资源，有剩余的话还会唤醒之后的队友。
* 那么问题就来了，假如老大用完后释放了5个资源，而老二需要6个，老三需要1个，老四需要2个。老大先唤醒老二，老二一看资源不够，他是把资源让给老三呢，还是不让？
* 答案是否定的！老二会继续park()等待其他线程释放资源，也更不会去唤醒老三和老四了。
* 在独占模式下，同一时刻只有一个线程去执行，这样做未尝不可；但共享模式下，多个线程是可以同时执行的，现在因为老二的资源需求量大，而把后面量小的老三和老四也都卡住了。
* 当然，这并不是问题，只是AQS保证严格按照入队顺序唤醒罢了（保证公平，但降低了并发）。
    ```java
        private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
    ```

##### 小结
* tryAcquireShared()尝试获取资源，成功则直接返回；
* 失败则通过doAcquireShared()进入等待队列park()，直到被unpark()/interrupt()并成功获取到资源才返回。整个等待过程也是忽略中断的。
* 此方法在setHead()的基础上多了一步，就是自己苏醒的同时，如果条件符合（比如还有剩余资源），还会去唤醒后继结点

#### 主要方法的源码实现-共享资源释放
releaseShared(int)方法是共享模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果成功释放且允许唤醒等待线程，它会唤醒等待队列里的其他线程来获取资源。
##### releaseShared(int)
```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
````
* PS：独占模式下的`tryRelease()`在完全释放掉资源（`state=0`）后，才会返回true去唤醒其他线程，这主要是基于独占下可重入的考量；而共享模式下的`releaseShared()则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。

##### doReleaseShared()
```java
private void doReleaseShared() {
    /*
        * Ensure that a release propagates, even if there are other
        * in-progress acquires/releases.  This proceeds in the usual
        * way of trying to unparkSuccessor of head if it needs
        * signal. But if it does not, status is set to PROPAGATE to
        * ensure that upon release, propagation continues.
        * Additionally, we must loop in case a new node is added
        * while we are doing this. Also, unlike other uses of
        * unparkSuccessor, we need to know if CAS to reset status
        * fails, if so rechecking.
        */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

##### 小结
尽管过程为了并发性考虑了很多，其实还是将占有的资源进行释放就好了。这点和独占资源的释放很类似

## 总结
* AQS通过volatile的state变量以及其对应的原子性操作保证并发
* AQS严格保证执行顺序（公平锁），涉及到公平不公平（能不能插队），这直接影响到队列的吞吐（比如说state有5块举铁，但是排后面的老哥A想拿6块举铁，而他后面的老哥B想拿3块举铁，那现在是等到有6块举铁了给老哥A，还是直接给老哥B三块举铁呢？），因此也要考虑使用场景的问题。
* 另外好像也有非公平锁，有时间会再康康出一个笔记。