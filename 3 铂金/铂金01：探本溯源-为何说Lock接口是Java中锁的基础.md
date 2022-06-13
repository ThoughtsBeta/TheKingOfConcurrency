欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第14篇**。

在**黄金**系列中，我们介绍了并发中一些问题，比如死锁、活锁、线程饥饿等问题。在并发编程中，这些问题无疑都是需要解决的。所以，在铂金系列文章中，我们会从并发中的问题出发，探索Java所提供的**锁的能力**以及它们是如何解决这些问题的。

作为铂金系列文章的第一篇，我们将从**Lock接口**开始介绍，因为它是Java中锁的基础，也是并发能力的基础。


## 一、理解Java中锁的基础：Lock接口

在青铜系列文章中，我们介绍了通过`synchronized`关键字实现对方法和代码块加锁的用法。**然而，虽然`synchronized`非常好用、易用，但是它的灵活度却十分有限，不能灵活地控制加锁和释放锁的时机**。所以，为了更灵活地使用锁，并满足更多的场景需要，就需要我们能够自主地定义锁。于是，就有了**Lock接口**。

理解Lock最直观的方式，莫过于直接在JDK所提供的并发工具类中找到它，如下图所示：

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/15/16237478266162.jpg)

可以看到，Lock接口提供了一些能力API，并有一些具体的实现，如ReentrantLock、ReentrantReadWriteLock等。

### 1. Lock的五个核心能力API

* `void lock()`：获取锁。**如果当前锁不可用，则会被阻塞直至锁释放**；
* `void lockInterruptibly()`：获取锁并允许被中断。**这个方法和`lock()`类似，不同的是，它允许被中断并抛出中断异常**。
* `boolean tryLock()`：尝试获取锁。**会立即返回结果，而不会被阻塞**。
* `boolean tryLock(long timeout, TimeUnit timeUnit)`：尝试获取锁并等待一段时间。这个方法和`tryLock()`，但是它会根据参数等待–会，**如果在规定的时间内未能获取到锁就会放弃**；
* `void unlock()`：释放锁。

### 2. Lock的常见实现

在Java并发工具类中，Lock接口有一些实现，比如：
* ReentrantLock：可重入锁；
* ReentrantReadWriteLock：可重入读写锁；

除了列举的两个实现外，还有一些其他实现类。对于这些实现，暂且不必详细了解，后面会详细介绍。**在目前阶段，你需要理解的是Lock是它们的基础**。

## 二、自定义Lock

接下来，我们基于前面的示例代码，看看如何将`synchronized`版本的锁用Lock来实现。

```java
 public static class WildMonster {
   private boolean isWildMonsterBeenKilled;
   
   public synchronized void killWildMonster() {
     String playerName = Thread.currentThread().getName();
     if (isWildMonsterBeenKilled) {
       System.out.println(playerName + "未斩杀野怪失败...");
       return;
     }
     isWildMonsterBeenKilled = true;
     System.out.println(playerName + "斩获野怪！");
   }
 }

```


### 1. 实现一把简单的锁

创建类WildMonsterLock并实现Lock接口，WildMonsterLock将是取代`synchronized`的关键：

```java
// 自定义锁
public class WildMonsterLock implements Lock {
    private boolean isLocked = false;

    // 实现lock方法
    public void lock() {
        synchronized (this) {
            while (isLocked) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            isLocked = true;
        }
    }
    
    // 实现unlock方法
    public void unlock() {
        synchronized (this) {
            isLocked = false;
            this.notify();
        }
    }
}

```

在实现Lock接口时，你需要实现它上述的所有方法。不过，为了简化代码方便展示，我们移除了WildMonsterLock类中的`tryLock`等方法。

对于`wait`和`notify`方法的时候，如果你不熟悉的话，可以查看青铜系列的文章。**这里需要提醒的是，`notify`在使用时务必要和`wait`是同一个监视器**。

基于刚才定义的WildMonsterLock，创建WildMonster类，并在方法killWildMonster中使用WildMonsterLock对象，从而取代synchronized.

```java
// 使用刚才自定义的锁
 public static class WildMonster {
   private boolean isWildMonsterBeenKilled;

   public void killWildMonster() {
     // 创建锁对象
     Lock lock = new WildMonsterLock(); 
     // 获取锁
     lock.lock(); 
     try {
       String playerName = Thread.currentThread().getName();
       if (isWildMonsterBeenKilled) {
         System.out.println(playerName + "未斩杀野怪失败...");
         return;
       }
       isWildMonsterBeenKilled = true;
       System.out.println(playerName + "斩获野怪！");
     } finally {
       // 执行结束后，无论如何不要忘记释放锁
       lock.unlock();
     }
   }
}
```
输出结果如下：

```shell
哪吒斩获野怪！
典韦未斩杀野怪失败...
兰陵王未斩杀野怪失败...
铠未斩杀野怪失败...

Process finished with exit code 0
```

从结果中可以看到：**只有哪吒一人斩获了野怪，其他几个英雄均以失败告终，结果符合预期**。这说明，WildMonsterLock达到了和`synchronized`一致的效果。

**不过，这里有细节需要注意**。在使用`synchronized`时我们无需关心锁的释放，JVM会帮助我们自动完成。然而，**在使用自定义的锁时，一定要使用`try...finally`来确保锁最终一定会被释放**，否则将造成后续线程被阻塞的严重后果。



### 2. 实现可重入的锁

在`synchronized`中，**锁是可以重入的**。**所谓锁的可重入，指的是锁可以被线程重复或递归调用**。比如，加锁对象中存在多个加锁方法时，当线程在获取到锁进入其中任一方法后，线程应该可以同时进入其他的加锁方法，而不会出现被阻塞的情况。当然，前提条件是这个加锁的方法用的是同一个对象的锁（监视器）。

在下面这段代码中，方法A和B都是同步方法，并且A中调用B. 那么，线程在调用A时已经获得了当前对象的锁，那么线程在A中调用B时可以直接调用，这就是锁的可重入性。

```java

public class WildMonster {
    public synchronized void A() {
        B();
    }
    
    public synchronized void B() {
       doSomething...
    }
}

```

所以，为了让我们自定义的WildMonsterLock也支持可重入，我们需要对代码进行一点改动。

```java
public class WildMonsterLock implements Lock {
    private boolean isLocked = false;
   
    // 重点：增加字段保存当前获得锁的线程
    private Thread lockedBy = null;
    // 重点：增加字段记录上锁次数
    private int lockedCount = 0;

    public void lock() {
        synchronized (this) {
            Thread callingThread = Thread.currentThread();
            // 重点：判断是否为当前线程
            while (isLocked && lockedBy != callingThread) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            isLocked = true;
            lockedBy = callingThread;
            lockedCount++;
        }
    }

    public void unlock() {
        synchronized (this) {
            // 重点：判断是否为当前线程
            if (Thread.currentThread() == this.lockedBy) {
                lockedCount--;
                if (lockedCount == 0) {
                    isLocked = false;
                    this.notify();
                }
            }
        }
    }
}
```
在新的WildMonsterLock中，我们增加了`this.lockedBy`和`lockedCount`字段，并在加锁和解锁时增加对线程的判断。**在加锁时，如果当前线程已经获得锁，那么将不必进入等待。而在解锁时，只有当前线程能解锁**。

`lockedCount`字段则是为了保证解锁的次数和加锁的次数是匹配的，比如加锁了3次，那么相应的也要3次解锁。

### 3. 关注锁的公平性

在黄金系列文章中，我们提到了线程在竞争中可能被饿死，因为竞争并不是公平的。**所以，我们在自定义锁的时候，也应当考虑锁的公平性**。

## 三、小结

以上就是关于Lock的全部内容。在本文中，我们介绍了Lock是Java中各类锁的基础。它是一个接口，提供了一些能力API，并有着完整的实现。并且，我们也可以根据需要自定义实现锁的逻辑。所以，在学习Java中各种锁的时候，最好先从Lock接口开始。同时，在替代synchronized的过程中，我们也能感受到Lock有一些synchronized所不具备的优势：

* **synchronized用于方法体或代码块，而Lock可以灵活使用，甚至可以跨越方法**；
* **synchronized没有公平性，任何线程都可以获取并长期持有，从而可能饿死其他线程。而基于Lock接口，我们可以实现公平锁，从而避免一些线程活跃性问题**；
* **synchronized被阻塞时只有等待，而Lock则提供了`tryLock`方法，可以快速试错，并可以设定时间限制，使用时更加灵活**；
* **synchronized不可以被中断，而Lock提供了`lockInterruptibly`方法，可以实现中断**。

另外，在自定义锁的时候，要考虑锁的公平性。而在使用锁的时候，则需要考虑锁的安全释放。

## 夫子的试炼

* 基于Lock接口，自定义实现一把锁。

## 延伸阅读与参考资料

* [Locks in Java](http://tutorials.jenkov.com/java-concurrency/locks.html)
* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。