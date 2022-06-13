欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第15篇**。

在上篇文章中，我们介绍了Java中锁的基础Lock接口。在本文中，我们将介绍Java中锁的另外一个重要的基本型接口，即**ReadWriteLock**接口。

**在探索Java中的并发时，ReadWriteLock无疑是重要的，然而理解它却并不容易**。如果你此前曾经检索资料，应该会发现大部分的文章对它的描述都比较晦涩难懂，或连篇累牍的源码陈列，或隔靴搔痒的三言两语，既说不到重点，也说不清来龙去脉。

**所以，在本文中我们会将介绍的重点放在对思路的理解上，而不是对源码的解读上**。对于源码以及其背后的知识，我们将在后面的更高级的系列中进行讲解。

## 一、理解ReadWriteLock存在的价值

理解ReadWriteLock，首先要理解它存在的意义是什么。**换言之，它要解决什么问题**。为此，我们不妨从下图着手一探究竟。

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/17/16239049652812.jpg)

不知你看明白了没有，这幅图所表达的有三层含义：

* 大量线程在竞争同一份资源；
* 这些线程中有的是**读请求**，有的是**写请求**；
* 在多个线程的请求中，**读请求明显高于写请求**。

这样的场景是否似曾相识？**没错，它就是典型的缓存应用场景**。

众所周知，缓存的存在是为了提高应用的读写性能。一方面，我们需要通过缓存拦截大量的读数据的请求。另一方面，我们也需要不定期地更新缓存。**但总体而言，更新缓存的次数远远小于读缓存的次数**。

在这个过程中，**关键问题在于，为了保持数据一致性，我们在读写缓存的时候，不能让读请求拿到脏数据，这就需要用到锁**。然而，**更关键的问题在于，虽然读写之间需要互斥，但读与读之间不可以互斥**。

总结来说，这个问题主要有下面这几个要点：

* **数据允许多个线程同时读取，但只允许一个线程进行写入**；
* **在读取数据的时候，不可以存在写操作或者写请求**；
* **在写数据的时候，不可以存在读请求**。

如果你对此仍然有些迷茫，**那么下面这张图建议你收藏**，这张图正是ReadWriteLock对问题的概述和它的解决方案，也是诠释ReadWriteLock最好的一幅图。

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/17/16239049385732.jpg)


在你没有理解ReadWriteLock之前，你会觉得它十分晦涩且源码枯燥。**然而，一旦你理解它要解决的问题，以及它所提供的方案后，你会发现它的设计竟然如此巧妙**。它竟然设计了两种截然不同的锁，**其中一把正如我们此前认知的那样是线程互斥的，而另一把锁竟然可以为多个线程所共享**！两把锁的完美配合，解决了并发读写的场景问题。

在恍然大悟后，所谓源码不过是队列与共享，它们是ReadWriteLock的一种实现方式，而不是阻挡你理解的绊脚石。

## 二、自主实现ReadWriteLock

在理解了ReadWriteLock背后的问题和它的解决思路之后，我们就可以完全抛开JDK中的源码自己实现一把读写锁。

```java
public class ReadWriteLock{

  private int readers       = 0;
  private int writers       = 0;
  private int writeRequests = 0;

  public synchronized void lockRead() throws InterruptedException{
    while(writers > 0 || writeRequests > 0){
      wait();
    }
    readers++;
  }

  public synchronized void unlockRead(){
    readers--;
    notifyAll();
  }

  public synchronized void lockWrite() throws InterruptedException{
    writeRequests++;

    while(readers > 0 || writers > 0){
      wait();
    }
    writeRequests--;
    writers++;
  }

  public synchronized void unlockWrite() throws InterruptedException{
    writers--;
    notifyAll();
  }
}
```

在读锁`lockRead()`中，是不允许有**写请求**或**写操作**的。如果有，那么读请求将进入等待。

而在`lockWrite()`中，**同时不允许读请求和其他写操作的存在，此时只允许有一个写请求**。

以上就是读写锁简单的自主实现方式。**当然，它是不完善的，只是基本的示例**。它没有考虑到基本的**线程重入**问题，真实情况也比它复杂很多，但你理解它的意思就好。

## 三、Java中的ReadWriteLock是如何实现的

最后，我们再来看JDK中的ReadWriteLock实现的一些基本思路。ReadWriteLock和我们上篇所说的Lock接口以及其他类的基本关系如下图所示：


![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/17/16239040236528.jpg)

可以看到，JDK中的读写锁的实现是在**ReentrantReadWriteLock**这个类中。ReentrantReadWriteLock包含了两个内部类：ReadLock和WriteLock，而这两个类又实现了Lock接口。


**读写锁的升级与降级**

读写锁的升级与降级是ReentrantReadWriteLock中的一个重要知识点，也是高频的面试题。

**从读锁到写锁，称之为锁的升级，反之为锁的降级**。理解读写锁的升级和降级，最直观的方式是写代码验证。

代码片段1，先获取读锁，再获取写锁。

```java
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
        readWriteLock.readLock().lock();
        System.out.println("已经获取读锁...");
        readWriteLock.writeLock().lock();
        System.out.println("已经获取写锁...");
    }
}
```
输出结果如下：

```shell
已经获取读锁...
```

代码片段2，先获取写锁，再获取读锁：

```java
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
        readWriteLock.writeLock().lock();
        System.out.println("已经获取写锁...");
        readWriteLock.readLock().lock();
        System.out.println("已经获取读锁...");
    }
}
```
输出结果如下：

```shell
已经获取写锁...
已经获取读锁...

Process finished with exit code 0
```

这样一来，结果已经十分明了。**ReentrantReadWriteLock支持锁的降级，但不支持锁的升级**。

**读写锁中的公平性**

在前面的文章中，我们讲过线程饥饿的由来和后果，所以良好的并发工具类在设计时都会考虑到公平性，ReentrantReadWriteLock也是如此。

**在ReentrantReadWriteLock中，同时提供了公平和非公平两种模式，且默认为非公平模式**。从下面摘取的源码片段中，可以清晰地看到。


```java
 public ReentrantReadWriteLock() {
        this(false);
 }

    /**
   /**
 * Creates a new {@code ReentrantReadWriteLock} with
 * default (nonfair) ordering properties.
 */
public ReentrantReadWriteLock() {
  this(false);
}

/**
 * Creates a new {@code ReentrantReadWriteLock} with
 * the given fairness policy.
 *
 * @param fair {@code true} if this lock should use a fair ordering policy
 */
public ReentrantReadWriteLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
  readerLock = new ReadLock(this);
  writerLock = new WriteLock(this);
}
```

## 小结

以上就是关于读写锁的全部内容。在本文中，我们从缓存问题出发，接着从ReadWriteLock中寻找答案，以便能从更轻松的角度理解ReadWriteLock的来龙去脉。

理解ReadWriteLock的关键不在于对源码的剖析，而在于对其思路的理解。

另外，我们简单地介绍了ReentrantReadWriteLock中的一些关键知识点，但诸如其背后的AQS等并没有展开陈述。对此也不必着急，我们会在后面有详细的分析介绍。



正文到此结束，恭喜你又上了一颗星✨

## 夫子的试炼

* 尝试在示例代码中增加对读写线程的重入支持。

## 延伸阅读与参考资料

* [示例代码参考](http://tutorials.jenkov.com/java-concurrency/read-write-locks.html)
* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。

