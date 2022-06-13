欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第2篇**。

在前面的《兵分三路：如何创建多线程》文章中，我们已经通过Thread和Runnable直观地了解**如何在Java中创建一个线程**，相信你已经有了一定的体感。在本篇文章中，我们将基于前面的示例代码，对线程做些必要的说明，以帮助你从更基础的层面认知线程，并为后续的学习打下基础。

## 一、从进程认知线程

在上世纪的80年代中期之前，**进程**一直都是操作系统中**拥有资源**和**独立运行**的基本单位。可是，随着计算机的发展，人们对操作系统的吞吐量要求越来越高，并且多处理器也逐渐发展起来，进程作为基本的调度单位已经越来越不合时宜，因为它太重了。

想想看，我们在创建进程的时候，需要给它们创建PCB，并且还要分配需要的所有资源，如内存空间、I/O设备等等。然而，在进程切换时，系统还需要保留当前进程的CPU环境并设置新的进程CPU环境，这些都是需要花费时间的。也就是说，在**进程的创建、切换以及销毁的过程中，系统要花费巨大的开销**。如此，在一个操作系统中，就不能设置过多数量的进程，并且还不能频繁切换。显然，这不符合时代的发展需要了。

因此，进程**切换的巨大开销**和**多核CPU**的发展，线程便顺势而生。

从概念上，线程可以理解为**它是操作系统中独立运行的基本单元，一个进程可以拥有多个线程**。并且，**和进程相比，线程拥有进程很多相似属性，因此线程有时候也被称为轻量级进程（Light-Weight Process）**。

**为什么线程相对轻量**

在线程之前，系统切换任务时需要切换进程，开销巨大。然而，引入线程后，线程隶属于进程，**进程仍是资源的拥有者，线程只占据少量的资源**。同时，线程的切换并不会导致进程的切换，因此开销较小。此外，进程和线程都可以并发执行，操作系统也因此获得了更好的并发性，也能有效地提高系统资源的利用率和系统吞吐量。

## 二、Thread类-Java中的线程基础

在本文的第一部分，我们从**操作系统**层面认识了**线程**。而在Java中，我们则需要通过**Thread**类认知线程，包括它的一些基本属性和基本方法。Thread是Java多线程基础中的基础，请不要因为它简单就忽略这部分的内容。

下面的这幅图概括展示了Thread类的核心属性和方法：

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/05/17/16212386797295.jpg)

### 1. Thread中如何构造线程
Thread中共有9个公共构造器，当然我们不用掌握全部的构造，熟悉其中几个比较常用的即可：
* `Thread()`，这个构造器会默认生成一个`Thread-`+`n`的名字，`n`是由内部方法`nextThreadNum()`生成的一个整型数字；
* `Thread(String name)`，在构建线程时指定线程名，是一个很不错的实践；
* ` Thread(Runnable target) `，传入`Runnable`的实例，这个我们在上一篇文章中已经展示过；
* `Thread(Runnable target, String name)`，在传入`Runnable`实例时指定线程名。


### 2. Thread中的关键属性
**从`init()`方法理解Thread的构造**

虽然Thread有9个构造函数，但最终都是通过下面的这个`init()`方法进行构造，所以了解了这个方法，就了解了Thread的构造过程。
```Java

    /**
     * Initializes a Thread.
     *
     * @param g the Thread group
     * @param target the object whose run() method gets called
     * @param name the name of the new Thread
     * @param stackSize the desired stack size for the new thread, or
     *        zero to indicate that this parameter is to be ignored.
     * @param acc the AccessControlContext to inherit, or
     *            AccessController.getContext() if null
     * @param inheritThreadLocals if {@code true}, inherit initial values for
     *            inheritable thread-locals from the constructing thread
     */
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals)
```

`init()`方法几十行的代码，不过为了节省篇幅，我们此处只贴出了方法的签名，具体的方法体内容可以自行去看。在方法的签名中，有几个重要的参数：
* `g`：线程所属的线程组。线程组是一组具有相似行为的线程集合；
* `target`：继承`Runnable`的对象实例；
* `name`：线程的名字；

其他几个参数通常不需要自定义，保持默认即可。如果你翻看`init()`的方法体代码，可以看到`init()`对下面几个属性做了初始化：
* `name`：线程的名字；
* `group`：线程所属的线程组；
* `daemon`：是否为守护进程；
* `priority`：线程的优先级；
* `tid`：线程的ID；

#### 关于名字
虽然Thread默认会生成一个线程名，但为了方便日志输出和问题排查，比较建议你在创建线程时自己手动设置名称，比如`anQiLaPlayer`的线程名可以设置为`Thread-anQiLa`。

#### 关于线程ID
和线程名一样，每个线程都有自己的ID，如果你没有指定的话，Thread会自动生成。确切地说，线程的ID是根据`threadSeqNumber()`对Thread的静态变量`threadSeqNumber`进行累加得到：
```Java
private static synchronized long nextThreadID() {
    return ++threadSeqNumber;
}

```

#### 关于线程优先级
在创建新的线程时，线程的优先级默认和当前父线程的优先级一致，当然我们也可以通过`setPriority(int newPriority)`方法来设置。不过，在设置线程优先级时需要注意两点：
* **Thread线程的优先级设置是不可靠的**：我们可以通过数字来指定线程调度时的优先级，然而最终执行时的调度顺序将由操作系统决定，因为Thread中的优先级设置并不是和所有的操作系统一一对应；
* **线程组的优先级高于线程优先级**：每个线程都会有一个线程组，我们所设置的线程优先级数字不能高于线程组的优先级。如果高于，将会直接使用线程组的优先级。

### 3. 线程中的关键方法

Thread中几个重要的方法，如`start()`、`join()`、`sleep()`、`yield()`、`interrupted()`等，关于这几种方法的用法，我们会在下一篇文章中结合线程的状态进行讲解。**需要注意的是，`notify()`、`wait()`等并不是Thread类的方法，它们是Object的方法**。

## 三、多线程的应用场景

通过前面的分析，我们已经从操作系统层面和Java中认知了线程。那么，什么样的场景需要考虑使用多线程？

总的来说，当你遇到以下两类场景时，需要考虑多线程：

**1. 异步**

当两个**独立逻辑单元**不需要同步顺序完成时，可以通过多线程异步处理。

比如，用户注册后发送邮件消息。很显然，**注册**和**发送消息**是两个独立逻辑单元，在注册完成后，我们可以另起线程完成消息的发送，从而实现逻辑解耦并缩短注册单元的响应时间。

**2. 并发**
现在的计算机基本都是多核处理器，在处理批量任务时，可以通过多线程提高处理速度。

比如，假设系统需要向100万的用户发送消息。可以想象，如果单线程处理不知道猴年马月才能完成。而此时，我们便可以通过线程池创建多线程大幅提高效率。

注意，对于一些同学来说，你可能还没有接触过多线程的应用场景。但是，请不要因为工作中的场景简单或数据量较低就忽视多线程的应用，多线程在你身边的各类中间件和互联网大厂中都有着极为广泛的应用。

**应用多线程时的风险提示**

虽然多线程有很多的好处，但仍然要根据场景客观分析，对多线程不合理的使用会增加系统的**安全风险**和**维护风险**。所以，在使用多线程时，请务必确认**场景的合理性**，以及它在你技术能力掌控之中。

以上就是文本的全部内容，恭喜你又上了一颗星✨

## 夫子的试炼

* 使用不同的构造方式，编写两个线程并打印出线程的关键信息；
* 检索资料，详细比对进程与线程的区别。

## 延伸阅读与参考资料

* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。

