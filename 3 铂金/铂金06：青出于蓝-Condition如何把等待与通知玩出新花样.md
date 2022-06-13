欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第19篇**。

在上一篇文章中，我们介绍了阻塞队列。如果你阅读过它的源码，那么你一定会注意到源码有两个Condition类型的变量：`notEmpty`和`notFull`，在读写队列时你也会注意到它们是如何被使用的。事实上，在使用JUC中的各种锁时，Condition都很有用场，你很有必要了解它。所以，本文就为你介绍它的来龙去脉和用法。

在前面的系列文章中，我们多次提到过`synchronized`关键字，相信你已经对它的用法了如于心。在多线程协作时，有两个另外的关键字经常和`synchronized`一同出现，它们相互配合，就是`wait`和`notify`，比如下面的这段代码：

```java
public class CountingSemaphore {
  private int signals = 0;
  public synchronized void take() {
    this.signals++;
    this.notify();  // 发送通知
  }
  public synchronized void release() throws InterruptedException {
    while (this.signals == 0)
      this.wait(); // 释放锁，进入等待
	    this.signals--;
  }
}
```

`synchronized`是Java的原生同步工具，`wait`和`notify`是它的原生搭档。然而，在铂金系列中，我们已经介绍了Lock接口和它的一些实现，比如可重入锁ReentrantLock等。相比于`synchronized`，JUC所封装的这些锁工具在功能上要丰富得多，也更加容易使用。所以，相应的配套自然也要跟上，于是Condition就应运而生。

比如在上文的阻塞队列中，Condition就已经闪亮登场：

```java
public class LinkedBlockingQueue < E > extends AbstractQueue < E >
    implements BlockingQueue < E > , java.io.Serializable {

        ...省略源码若干

        // 定义Condition
        // 注意，这里定义两个Condition对象，用于唤醒不同的线程
        private final Condition notEmpty = takeLock.newCondition();
        private final Condition notFull = putLock.newCondition();

        public E take() throws InterruptedException {
            E x;
            int c = -1;
            final AtomicInteger count = this.count;
            final ReentrantLock takeLock = this.takeLock;
            takeLock.lockInterruptibly();
            try {
                while (count.get() == 0) {
                    // 进入等待
                    notEmpty.await();
                }
                x = dequeue();
                c = count.getAndDecrement();
                if (c > 1)
                    notEmpty.signal();
            } finally {
                takeLock.unlock();
            }
            if (c == capacity)
                signalNotFull();
            return x;
        }

        private void signalNotEmpty() {
            final ReentrantLock takeLock = this.takeLock;
            takeLock.lock();
            try {
                // 发送唤醒信号
                notEmpty.signal();
            } finally {
                takeLock.unlock();
            }
        }
        ...省略源码若干

    }
```

**从功能定位上说，作为Lock的配套工具，Condition是`wait`、`notify`和`notifyAll`增强版本，`wait`和`notify`有的能力它都有，`wait`和`notify`没有的能力它也有**。

JUC中的Condition是以接口的形式出现，并定义了一些核心方法：

* `await()`：让当前线程进入等待，直到收到信号或者被中断；
* `await(long time, TimeUnit unit)`：让当前线程进入等待，直到收到信号或者被中断，或者到达指定的等待超时时间；
* `awaitNanos(long nanosTimeout)`：让当前线程进入等待，直到收到信号或者被中断，或者到达指定的等待超时时间，只是在时间单位上和上一个方法有所区别；
* `awaitUninterruptibly(）`：**让当前线程进入等待，直到收到信号。注意，这个方法对中断是不敏感的**；
* `awaitUntil(Date deadline)`：**让当前线程进入等待，直到收到信号或者被中断，或者到达截止时间**；
* `signal()`：随机唤醒一个线程；
* `signalAll()`：唤醒所有等待的线程。


**从Condition的核心方法中可以看到，相较于原生的通知与等待，它的能力明显增强了很多，比如`awaitUninterruptibly()`和`awaitUntil()`。另外，Condition竟然是可以唤醒指定线程的，这就很有意思**。

作为接口，我们并不需要手动实现Condition，JUC已经提供了相关的实现，你可以在ReentrantLock中直接使用它。相关的类、接口之间的关系如下所示：

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/27/16247728628301.jpg)


## 小结

以上就是关于Condition的全部内容。Condition并不复杂，它是JUC中Lock的配套，在理解时要结合原生的`wait`和`notify`去理解。关于Condition与它们之间的详细区别，已经都在下面的表格里：

|对比项|Object's Monitor methods| Condition
|---|---|---|
|**前置条件**|获取对象的锁|调用Lock获取锁，调用lock.newCondition()获取Condition对象|
|**调用方式**|直接调用，如object.wait()|直接调用，如condition.await()|
|**等待队列个数**|一个|**多个**|
|**当前线程释放锁并进入等待状态**|✔︎|✔︎|
|**当前线程释放锁并进入等待状态，在等待时不响应中断**|✘|**✔︎**|
|**当前线程释放锁并进入超时等待**|✔︎|✔︎|
|**当前线程释放锁并进入等待到未来某个时刻**|✘|**✔︎**|
|**唤醒等待队列中的某一个线程**|✔︎|✔︎|
|**唤醒等待队列中的全部线程**|✔︎|✔︎|

理解表格中的各项差异，不要死记硬背，而是要基于Condition接口中定义的方法，从关键处理解它们的不同。

正文到此结束，恭喜你又上了一颗星✨

## 夫子的试炼

* 编写代码使用Condition唤醒指定线程。

## 延伸阅读与参考资料

* 小结表格中的内容由网络图片提取，未能找到原始出处，知道的请评论告知，感谢！

* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。

