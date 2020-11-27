---
layout: post
title:  "并发编程 - Compare And Swap"
date:   2020-11-24 12:20:30 +0800
categories: concurrent
---

* TOC
{:toc}

原子变量的实现原理基于 CPU 提供的 Compare And Swap 指令。

```java
public final int incrementAndGet() {    
    for (;;) {
        // 获取当前值
        int current = get();

        // 计算目标值
        int next = current + 1;

        // 再次获取当前值，如果当前值与之前获取的 current 值一致，那么将当前值修改为目标值
        if (compareAndSet(current, next)) {
            return next;
        }
    }
}

public final bool compareAndSet(int old, int next) {
    // 以下代码是原子操作，由 CPU 提供的指令完成

    // 再次获取当前值
    int current = get();
    if (current != old) {
        // 当前值被其他线程修改过了，不能直接设置，因此返回 false
        return false;
    }

    // 当前值没有被其它线程修改过，因此可以修改
    set(next);
    return true;
}
```

可以看到原子操作当中包含了一个循环，它不断检查值有没有被其他线程修改，只有在没有修改的情况下才把这个值修改为目标值，这样就保证了对这个值的修改是原子的，不会出现两个线程同时修改一个值的情况。

## 性能问题

根据上面的原理，原子操作在并发量高，或者说对变量的修改比较频繁的情况下可能会有问题，因为每次 CompareAndSwap 的时候，都检查到变量被其他线程修改了，然后 CompareAndSwap 一直失败，就会导致当前线程循环很多次。

比如下面的代码:

```java
package tech.dada;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * @author dongyu
 */
public class Application {

    public static AtomicInteger atomicInteger = new AtomicInteger();

    public static void main(String[] args) throws InterruptedException {
        // 线程一
        new Thread(() -> {
            long beginMillis = System.currentTimeMillis();
            for (int i = 0; i < 100000; ++i) {
                atomicInteger.incrementAndGet();
            }
            System.out.println("thread1 cost: " + (System.currentTimeMillis() - beginMillis));
        }).start();

        // 线程二
        new Thread(() -> {
            long beginMillis = System.currentTimeMillis();
            for (int i = 0; i < 100000; ++i) {
                atomicInteger.incrementAndGet();
            }
            System.out.println("thread2 cost: " + (System.currentTimeMillis() - beginMillis));
        }).start();

        Thread.sleep(3000);
        System.out.println("result: " + atomicInteger.get());
    }
}
```

如果两个线程同时跑，各自的耗时是 8ms 左右, 如果只有一个线程在跑，耗时是 3ms 左右。

不过工作中很难遇到修改频率这么高的情况，程序毕竟还要跑别的逻辑，执行其他指令，所以一般这种性能影响可以忽略。如果频率真的特别高(十万级别)，再来考虑这方面的影响。

## ABA 问题

ABA问题是无锁结构实现中常见的一种问题，可基本表述为：
- 进程P1读取了一个数值A
- P1被挂起(时间片耗尽、中断等)，进程P2开始执行
- P2修改数值A为数值B，然后又修改回A
- P1被唤醒，比较后发现数值A没有变化，程序继续执行。

对于P1来说，数值A未发生过改变，但实际上A已经被变化过了，继续使用可能会出现问题。

从 [维基百科](https://zh.wikipedia.org/wiki/%E6%AF%94%E8%BE%83%E5%B9%B6%E4%BA%A4%E6%8D%A2) 上看到了一个具体的例子来解释这种问题可能导致的错误：

有一个通过链表构造的栈，当前栈顶指针指向 0x0014 这个地址。

```
   top
    |
    V   
  0x0014
| Node A | --> |  Node X | --> ……
```

现在线程 P1 想要把 Node A 从栈顶 pop 出来，因此借助 CAS 实现了以下 pop 函数:

```
pop()
{
  // 借助 CAS 把栈顶指针指向下个节点
  do{
    ptr = top;            // ptr = top = NodeA
    next_prt = top->next; // next_ptr = NodeX
  } while(CAS(top, ptr, next_ptr) != true);
  return ptr;   
}
```

但是在 P1 执行 CAS 函数之前，另一个线程 P2 做了 ABA 的操作，先把 NodeA pop 出去，然后又 push 了两个节点 NodeB 和 NodeC。在这个过程中 P2 做了内存申请和释放的操作，由于内存重用，很不巧目前的栈顶 NodeC 地址仍然是 0x0014：

```
   top
    |
    V  
  0x0014
| Node C | --> | Node B | --> |  Node X | --> ……
```

这个时候 P1 的 CAS 函数执行，它发现 top 指针仍然指向 0x0014，它认为这个值没有被修改过，所以可以继续执行，因此就把 top 指向了 NodeX:

```
                                   top
                                    |
   0x0014                           V
 | Node C | --> | Node B | --> |  Node X | --> ……
```

这样就有问题了，实际上 P1 线程操作的时候应该把 top 指针指向 NodeB。

可以看到虽然把 top 设置为 top->next 这个操作是原子的，但是由于 ABA 问题，导致 top 和 top->next 原来的关系无效了，把错误的 top-next 设置给了 top。

最开始的 incrementAndGet() 例子不会有这个问题，因为 current 和 next 之间的关系始终是 +1, 在操作过程中值有没有修改过是没关系的。但是在这个链表的例子里 top 和 top->next 关系可能会变，比如一开始是 0x0014 -> 0x0020, 后来却变成了 0x0014 -> 0x0030。


## ABA 问题的解决方法

JDK 中的 AtomicStampedReference 类给数值增加了一个版本号，来解决 ABA 问题。总的来说，就是每次进行 CompareAndSet 操作时，不仅检查当前值跟旧值是否一致，还会检查版本号是否一致，只有在两者都一致的情况下，才更新值和版本号。

我们之前提过 incrementAndGet() 是不会有 ABA 问题的，因此利用这种方式来修改版本号，然后再用版本号来保障对目标值的修改不受 ABA 问题的影响。

```java
package tech.dada;

import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * @author dongdada
 */
public class Application {

    private static AtomicStampedReference<Integer> reference = new AtomicStampedReference<Integer>(127, 1);

    public static void main(String[] args) throws InterruptedException {
        // 线程一
        new Thread(() -> {
            // 先睡一秒，等线程二开始执行
            try { Thread.sleep(1000); } catch (InterruptedException e) {}

            // 如果当前值是 127, 就将其修改为 15
            // 如果当前时间戳是 n, 就将其修改为 n+1
            reference.compareAndSet(127, 15, reference.getStamp(), reference.getStamp() + 1);

            // 如果当前值是 15, 就将其修改为 127
            // 如果当前时间戳是 n, 就将其修改为 n+1
            reference.compareAndSet(15, 127, reference.getStamp(), reference.getStamp() + 1);
        }).start();

        // 线程二
        new Thread(() -> {
            // 先获取到当前时间戳
            int stamp = reference.getStamp();

            // 睡上 2 秒，等线程一完成 ABA 操作
            try { Thread.sleep(1000); } catch (InterruptedException e) {}

            // 此时由于时间戳是老版本的，因此 compareAndSet 会失败，这就避免了 ABA 发生时，值被不正确地修改的问题
            boolean result = reference.compareAndSet(127, 11, stamp, stamp + 1);
            System.out.println("线程二操作结果: " + result);
        }).start();

        Thread.sleep(3000);
    }
}
```