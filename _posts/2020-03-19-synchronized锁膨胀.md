---
layout:     post
title:      synchronized锁膨胀
subtitle:   无锁-偏向锁-轻量级锁-重量级锁
date:       2020-03-19
author:     LG
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - 并发
    - java
    
---



## synchronized锁膨胀


### 1. 基本概念

#### Java对象头

Java对象的对象头信息中的 Mark Word 主要是用来记录对象的锁信息的。

现在看一下 Mark Word 的对象头信息。如下：

![](https://tva1.sinaimg.cn/large/008eGmZEgy1gnta1s7p34j31400a80xn.jpg)

其实可以根据 mark word 的后3位就可以判断出当前的锁是属于哪一种锁。注意：表格中的正常锁其实就是无锁状态了。


### 2. 几种锁以及原理

#### 无锁

正常创建的对象，状态为无锁。对象头的Mark Word 中主要记录了 对象的年龄，也就是经历了多少次GC还存活下来。

#### 偏向锁

对象头的Mark Word 中记录的信息比正常锁多的是 记录了线程的信息，也就是线程的id。偏向锁，字面意思就是比较偏心，偏向于某个线程。

**偏向锁加锁原理**

经过分析可以看到，thread 字段刚开始的时候为0。工作原理是 cas(thread,0,当前线程), 也就是判断当前的 thread的值如果是0的话，就赋值为当前线程。如果cas 拿锁失败了，那么有俩种情况。

- 重入

  当前线程已经是偏向锁的持有者了，那么只需要判断 thread 字段是否就是当前线程，如果是的话，其实就是当前线程已经拿到这把锁了。那么此时就会在栈中创建lockRecord。因为栈是线程私有的。lockRecord 其实包含俩部分的内容，第一部分内容和锁对象的 markword中的对象是完全一样的。记为M区。第二部分是记录了当前对象，也就是加锁对象的指针。记为O区。在第一次加锁的时候，是不需要创建 lockRecord的。只有在重入的时候，才需要创建 lockRecord的。

- 其它线程持有锁

  那么此时就会升级为轻量级锁。

**偏向锁解锁原理**

解锁过程非常简单，只需要在当前线程的栈上删除最近的lockRecord对象就可以了。因为重入的时候是不断的创建这些lockRecord对象的。


#### 轻量级锁

主要是将锁对象的Mark Word更新为指向Lock Record的指针，也就是锁记录。注意，这个锁记录是在栈上的。

**轻量级锁加锁原理**

1. 线程在自己的栈桢中创建锁记录 LockRecord。
   ![](https://tva1.sinaimg.cn/large/008eGmZEgy1gnta5c8croj31ok0q6wki.jpg)

2. 将锁对象的对象头中的MarkWord复制到线程的刚刚创建的锁记录中。

3. 将锁对象的对象头的MarkWord替换为指向锁记录的指针。也就是进行cas操作。cas(ptr, null, lockRecord）。也就是如果对象头的ptr指针如果为空，那么就赋值为当前的lockRecord对象。并将线程栈帧中的Lock Record里的owner指针指向Object的 Mark Word。如果赋值成功了，那么就可以认为加锁成功了。

![](https://tva1.sinaimg.cn/large/008eGmZEgy1gnta7ewsy6j31e90u0h1e.jpg)

  如果加锁失败了，也有俩种情况。

- 重入。

  就是ptr 指向的是另外的一个lockRecord对象，但是也是当前线程创建的。也就是在当前线程的栈上是可以找到这个lockRecord对象。首先在栈上检查是否可以找到这个锁对象。如果可以找到，就是重入。如果是重入，那么就需要将锁记录中的Owner指针指向锁对象。

- 其它线程持有锁

  那么就进入重量级锁。

**轻量级锁解锁原理**

释放锁的时候，从栈上进行扫描，从后往前 找到最近的lockRecord,删除。

#### 重量级锁

主要是指向一个monitor 对象。

[synchronized重量级锁分析](https://szuliguo.github.io/2020/04/05/synchronized%E9%87%8D%E9%87%8F%E7%BA%A7%E9%94%81%E5%88%86%E6%9E%90/)


### 3. 看一段程序，看一下锁是如何进行演变的。

```java
package org.example;



import sun.misc.Unsafe;

import java.lang.reflect.Field;
/**
 * @author frank wy170862@alibaba-inc.com
 * @date 2020-02-24
 */
public class A {
    public static void main(String[] args) throws Exception {
        test2();
    }
    /**
     * 一、正常创建的对象，状态为无锁，观察hashcode和age的变化。hashcode 的计算是惰性的。刚开始的时候，对象的hashcode是0。只有在计算了hashcode以后，才会将hashcode存储到对象头，下次取hashcode的时候，就可以直接从对象头中取出hashcode了。
     *
     * 锁状态：无锁，hashCode：0,age: 0
     * ---------------
     *
     * 运行hashcode方法，得到hashcode:648129364
     * 锁状态：无锁，hashCode：648129364,age: 0
     * ---------------
     *
     * [GC (System.gc()) [PSYoungGen: 6568K->1095K(76288K)] 6568K->1103K(251392K), 0.0017301 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
     * [Full GC (System.gc()) [PSYoungGen: 1095K->0K(76288K)] [ParOldGen: 8K->932K(175104K)] 1103K->932K(251392K), [Metaspace: 3084K->3084K(1056768K)], 0.0054386 secs] [Times: user=0.02 sys=0.01, real=0.01 secs]
     * 运行一次gc，obj的age+1
     * 锁状态：无锁，hashCode：648129364,age: 1
     * ---------------
     *  @throws Exception
     */
    private static void test1() throws Exception {
        Object a = new Object();
        printLockHeader(a);
        System.out.println("运行hashcode方法，得到hashcode:" + a.hashCode());;
        printLockHeader(a);
        System.gc();
        System.out.println("运行一次gc，obj的age+1");
        // sleep 1s 让gc完成，但是不一定能100%触发gc，可以配合添加运行参数 -XX:+PrintGCDetails，观察确实gc了
        Thread.sleep(1000);
        printLockHeader(a);
    }
    /**
     * 二、正常创建的对象，状态为无锁，无锁状态直接加锁会变成轻量锁
     *
     * 锁状态：无锁，hashCode：0,age: 0
     * ---------------
     *
     * 对a加锁后
     * 锁状态：轻量级锁，LockRecord地址：1c00010c6e28
     * ---------------
     * @throws Exception
     */
    private static void test2() throws Exception {
        Object a = new Object();
        printLockHeader(a);
        synchronized (a){
            System.out.println("对a加锁后");
            printLockHeader(a);
        }
    }
    /**
     * 三、程序启动一定时间后，正常创建的对象，状态为偏向锁且thread为0，此时加锁默认为偏向锁
     * 一段时间一般是几秒，-XX:BiasedLockingStartupDelay=0可以指定默认就使用偏向锁，而不是无锁
     *
     * 锁状态：偏向锁，thread：0,epoch: 0,age: 0
     * ---------------
     *
     * 对a加锁后
     * 锁状态：偏向锁，thread：137069895700,epoch: 0,age: 0
     * ---------------
     *
     * 偏向锁重入后
     * 锁状态：偏向锁，thread：137069895700,epoch: 0,age: 0
     * ---------------
     * @throws Exception
     */
    private static void test3() throws Exception {
        Thread.sleep(5*1000);
        Object a = new Object();
        printLockHeader(a);
        synchronized (a){
            System.out.println("对a加锁后");
            printLockHeader(a);
            System.out.println("偏向锁重入后");
            synchronized (a){
                printLockHeader(a);
            }
        }
    }
    /**
     * 四、基于三，当另一个线程尝试使用对象锁的时候，升级为轻量锁
     *
     * 锁状态：偏向锁，thread：0,epoch: 0,age: 0
     * ---------------
     *
     * 线程1对a加锁后
     * 锁状态：偏向锁，thread：137122299998,epoch: 0,age: 0
     * ---------------
     *
     * 锁释放了
     * 线程2对a加锁后
     * 锁状态：轻量级锁，LockRecord地址：1c00015e6a18
     * ---------------
     * @throws Exception
     */
    private static void test4() throws Exception {
        Thread.sleep(5*1000);
        Object a = new Object();
        printLockHeader(a);
        new Thread(
                ()->{
                    synchronized (a){
                        System.out.println("线程1对a加锁后");
                        try {
                            printLockHeader(a);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    try {
                        Thread.sleep(10000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                }
        ).start();
        // 中间sleep1s，保证锁释放掉，使两个线程不会有竞争关系
        Thread.sleep(1000);
        System.out.println("锁释放了");
        new Thread(
                ()->{
                    synchronized (a){
                        System.out.println("线程2对a加锁后");
                        try {
                            printLockHeader(a);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
        ).start();
    }
    /**
     * 五、基于四，当产生竞争的时候偏向锁直接升级为重量级锁
     *
     * 锁状态：偏向锁，thread：0,epoch: 0,age: 0
     * ---------------
     *
     * 线程1对a加锁后
     * 锁状态：偏向锁，thread：137025283340,epoch: 0,age: 0
     * ---------------
     *
     * 线程2对a加锁后
     * 锁状态：重量级锁，Monitor地址：1fe758800a02
     * ---------------
     * @throws Exception
     */
    private static void test5() throws Exception {
        Thread.sleep(5*1000);
        Object a = new Object();
        printLockHeader(a);
        new Thread(
                ()->{
                    synchronized (a){
                        System.out.println("线程1对a加锁后");
                        try {
                            printLockHeader(a);
                            Thread.sleep(1000);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
        ).start();
        new Thread(
                ()->{
                    synchronized (a){
                        System.out.println("线程2对a加锁后");
                        try {
                            printLockHeader(a);
                            Thread.sleep(1000);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
        ).start();
    }
    /**
     * 六、四+五 演示偏向-轻量-重量过程
     *
     * 锁状态：偏向锁，thread：0,epoch: 0,age: 0
     * ---------------
     *
     * 线程1对a加锁后
     * 锁状态：偏向锁，thread：137272648594,epoch: 0,age: 0
     * ---------------
     *
     * 锁释放
     * 线程2对a加锁后
     * 锁状态：轻量级锁，LockRecord地址：1c0001b43e18
     * ---------------
     *
     * 锁释放
     * 线程3对a加锁后
     * 锁状态：轻量级锁，LockRecord地址：1c0001b43e18
     * ---------------
     *
     * 线程4对a加锁后
     * 锁状态：重量级锁，Monitor地址：1ff616600f02
     * ---------------
     * @throws Exception
     */
    private static void test6() throws Exception {
        Thread.sleep(5*1000);
        Object a = new Object();
        printLockHeader(a);
        new Thread(
                ()->{
                    synchronized (a){
                        System.out.println("线程1对a加锁后");
                        try {
                            printLockHeader(a);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    try {
                        Thread.sleep(10000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
        ).start();
        // 中间sleep1s，使线程不会有竞争关系
        Thread.sleep(1000);
        System.out.println("锁释放");
        new Thread(
                ()->{
                    synchronized (a){
                        System.out.println("线程2对a加锁后");
                        try {
                            printLockHeader(a);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }

                }
        ).start();
        // 中间sleep1s，使线程不会有竞争关系
        Thread.sleep(1000);
        System.out.println("锁释放");
        new Thread(
                ()->{
                    synchronized (a){
                        System.out.println("线程3对a加锁后");
                        try {
                            printLockHeader(a);
                            Thread.sleep(1000);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
        ).start();
        // 此时不再sleep，使线程必然发生竞争，升级为重量级锁
        new Thread(
                ()->{
                    synchronized (a){
                        System.out.println("线程4对a加锁后");
                        try {
                            printLockHeader(a);
                            Thread.sleep(1000);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
        ).start();
    }
    private static Unsafe getUnsafe() throws Exception {
        Class<?> unsafeClass = Class.forName("sun.misc.Unsafe");
        Field field = unsafeClass.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        return  (Unsafe) field.get(null);
    }
    private static void printLockHeader(Object obj) throws Exception {
        Unsafe us = getUnsafe();
        StringBuilder sb = new StringBuilder();
        int status = us.getByte(obj, 0L) & 0B11;
        // 0 轻量级 1 无锁或偏向 2 重量级 3 GC标记
        switch (status){
            case 0:
                // ptr_to_lock_record:62|lock:2
                long ptrToLockRecord =
                        (byteMod(us.getByte(obj, 0L))>>2) +
                                (byteMod(us.getByte(obj, 1L))<<6) +
                                (byteMod(us.getByte(obj, 2L))<<14) +
                                (byteMod(us.getByte(obj, 3L))<<22) +
                                (byteMod(us.getByte(obj, 4L))<<30) +
                                (byteMod(us.getByte(obj, 5L))<<38) +
                                (byteMod(us.getByte(obj, 6L))<<46) +
                                (byteMod(us.getByte(obj, 7L))<<54);
                sb.append("锁状态：轻量级锁，LockRecord地址：")
                        .append(Long.toHexString(ptrToLockRecord))
                ;
                break;
            case 1:
                boolean biased = (us.getByte(obj, 0L)&4) == 4;
                if(!biased){
                    // unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2
                    int hashCode = (int)(byteMod(us.getByte(obj, 1L))
                            + (byteMod(us.getByte(obj, 2L))<<8)
                            + (byteMod(us.getByte(obj, 3L))<<16)
                            + ((byteMod(us.getByte(obj, 4L))&Integer.MAX_VALUE) <<24))
                            ;
                    int age = (us.getByte(obj,0L)>>3)&0B1111;
                    sb.append("锁状态：无锁，hashCode：")
                            .append(hashCode)
                            .append(",age: ")
                            .append(age);
                }else{
                    //thread:54|epoch:2|unused:1| age:4 | biased_lock:1 | lock:2
                    long thread = (byteMod(us.getByte(obj, 1L))>>2) +
                            (byteMod(us.getByte(obj, 2L))<<6) +
                            (byteMod(us.getByte(obj, 3L))<<14) +
                            (byteMod(us.getByte(obj, 4L))<<22) +
                            (byteMod(us.getByte(obj, 5L))<<30) +
                            (byteMod(us.getByte(obj, 6L))<<38) +
                            (byteMod(us.getByte(obj, 7L))<<46);
                    ;
                    int epoch = us.getByte(obj, 1L) & 0B11;
                    int age = (us.getByte(obj,0L)>>3)&0B1111;
                    sb.append("锁状态：偏向锁，thread：")
                            .append(thread)
                            .append(",epoch: ")
                            .append(epoch)
                            .append(",age: ")
                            .append(age);
                }
                break;
            case 2:
                // ptr_to_heavyweight_monitor:62| lock:2
                long ptrToMonitor =
                        (byteMod(us.getByte(obj, 0L))>>2) +
                                (byteMod(us.getByte(obj, 1L))<<6) +
                                (byteMod(us.getByte(obj, 2L))<<14) +
                                (byteMod(us.getByte(obj, 3L))<<22) +
                                (byteMod(us.getByte(obj, 4L))<<30) +
                                (byteMod(us.getByte(obj, 5L))<<38) +
                                (byteMod(us.getByte(obj, 6L))<<46) +
                                (byteMod(us.getByte(obj, 7L))<<54);
                sb.append("锁状态：重量级锁，Monitor地址：")
                        .append(Long.toHexString(ptrToMonitor))
                ;
                break;
            case 3:
                sb.append("锁状态：GC标记");
                break;
            default:
                break;
        }
        if(obj instanceof Object[]){
            int arrLen = us.getInt(obj, 3L);
            sb.append("对象为数组类型，数组长度:")
                    .append(arrLen);
        }
        sb.append("\n").append("---------------").append("\n");
        System.out.println(sb.toString());
    }
    private static long byteMod(byte b){
        if(b>=0){
            return b;
        }
        return b + 256;
    }
}


```

### 4. 总结

刚开始，程序开始运行的时候，创建的对象都属于无锁对象。程序运行一段时间后，一段时间一般指的是4秒钟，创建的对象属于偏向锁对象。无锁状态下直接加锁会变为轻量级锁。偏向锁状态下的对象，加锁的话，会对当前线程有一个偏向。如果此时再有另外的一个线程过来申请锁，那么就会升级为轻量级锁对象。轻量级锁下，如果存在竞争，那么一定会升级为重量级锁。当然，偏向锁状态下，如果存在竞争，也会升级为重量级锁。

总结起来就是:如果存在竞争，那么一定会升级为重量级锁。如果存在另外一个线程想要拿到这把锁，就会升级为轻量级锁。偏向锁和轻量级锁的区别是：轻量级锁是可以另外一个线程来拿锁，也就是一个线程释放掉锁以后，另外一个线程可以来拿锁，这就是轻量级锁。如果一个线程释放掉锁以后，另外一个线程拿不到这把锁，那么就属于偏向锁。



## 鸣谢
[java 偏向锁、轻量级锁及重量级锁synchronized原理](https://www.cnblogs.com/deltadeblog/p/9559035.html)

[synchronized和锁优化](https://juejin.im/entry/5acde01a51882555867fc924
)