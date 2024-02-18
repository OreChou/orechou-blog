---
title: JUC 工具：LockSupport
date: 2023-06-09 17:00:00
tags: [Java, JUC]
---

# 简介
`LockSupport` 是 Java 并发工具包 `java.util.concurrent` 中的一个用于创建锁和其他同步类的基本线程阻塞原语。`LockSupport` 提供了 `park()` 和 `unpark()` 方法来停止和恢复线程。这些方法比低级的 `wait/notify` 和 `notifyAll` 方法更容易使用，并且与 `java.util.concurrent` 的高级同步工具更好地互相配合。

# 示例
在下面这个例子中，我们定义了两个线程。t1 打印数字，t2 打印字母。我们想要实现的效果是  t1 和 t2 交替打印，即输出结果为 "1A2B3C4D5E6F7G"。使用 `LockSupport` 的 `park()` 和 `unpark()` 方法，我们可以精确地控制线程的执行顺序，实现线程间的同步。

```Java
import java.util.concurrent.locks.LockSupport;

public class LockSupportDemo {
    static Thread t1 = null, t2 = null;

    public static void main(String[] args) {
        char[] a1 = "1234567".toCharArray();
        char[] a2 = "ABCDEFG".toCharArray();

        t1 = new Thread(() -> {
            for (char c : a1) {
                System.out.print(c);
                LockSupport.unpark(t2); // 叫醒t2
                LockSupport.park(); // 阻塞t1
            }
        }, "t1");

        t2 = new Thread(() -> {
            for (char c : a2) {
                LockSupport.park(); // 阻塞t2
                System.out.print(c);
                LockSupport.unpark(t1); // 叫醒t1
            }
        }, "t2");

        t1.start();
        t2.start();
    }
}
```

# 答疑
## Q1：park 方法中的 blocker 的作用
这个 blocker 参数实际上是一个与该挂起操作关联的对象，这个对象会在调用 `park()` 方法后被设置到该线程，用于后续诊断和监控工具查看。它并不参与 `park()` 方法的工作逻辑，也不会影响挂起或者恢复线程的操作。这个对象通常是执行该操作的同步抽象（例如锁）。在调用 `park` 方法之后，可以使用 `getBlocker(Thread)` 方法来检索和清除该对象。

总的来说，blocker 对象主要是用于诊断和监控目的，它能帮助开发者或者工具更好地理解线程为何被挂起，以及哪个对象是造成线程挂起的原因。

## Q2：LockSupport 相比于 Object.wait 和 Object.notify 的优势
`LockSupport` 相较于 `Object.wait` 和 `Object.notify` 提供了更高级、更安全、更灵活的线程同步机制，主要体现在以下几点：
1. 无需在同步代码块里：调用 `LockSupport` 的 `park()` 和 `unpark()` 方法不需要在 `synchronized` 代码块中，避免了不必要的同步，比较灵活。
2. 中断影响：如果线程正在调用 `park()` 阻塞，那么如果其他线程中断它，那么 `park()` 方法会返回，但不会抛出 `InterruptedException` 异常。
3. 提供了 blocker 的概念：`LockSupport` 可以知道是因为什么原因导致线程被阻塞，这对于问题排查等方面会更有帮助。
4. `unpark` 可以先于 `park` 调用：在 `Object.wait` 和 `Object.notify` 中，必须先执行 `wait` 再执行 `notify`，否则 `notify` 的作用会丢失，而 `LockSupport` 则允许 `unpark` 先执行。

## Q3: LockSupport.park()会释放锁资源吗
不会。`LockSupport.park()` 方法只是让当前线程进入等待状态，它不会释放任何线程可能持有的锁。这是因为，`LockSupport.park()` 是一个底层的线程阻塞工具，而不涉及 Java 的同步控制块（即 synchronized 关键字）或者 `java.util.concurrent` 包中的高级锁工具。
在使用 `LockSupport.park()` 时，需要特别注意这一点。如果一个线程在持有锁的情况下调用了 `park()` 方法并被阻塞，那么可能会导致其他试图获取该锁的线程也被阻塞，从而可能引发活跃性问题。