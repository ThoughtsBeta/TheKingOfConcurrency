欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第20篇**。

在上一篇文章中，我们介绍了Condition的用法。在本文中，将为你介绍CountDownLatch的用法。CountDownLatch是JUC中的一款常用工具类，当你在编写多线程代码时，**如果你需要协调多个线程的开始和结束动作时，它可能会是你的不错选择**。

## 一、CountDownLatch适用的两个典型应用场景

**场景1. 协调子线程结束动作：等待所有子线程运行结束**

对于资深的王者来说，下面这幅图一定再熟悉不过了。在王者开局匹配队友时，所有的玩家都必须进行确认操作，只有全部确认后才可以进入游戏，否则将进行重新匹配。**如果我们把各玩家看作是子线程的话，那么就需要各子线程完成确认动作，游戏才能继续**。其实，此类场景不止于王者，在生活中类似的场景还有很多。比如，所有乘客登机后飞机才能关舱门，除非超时后他们抛弃了你。

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/27/16248077942937.jpg)


对于上图所示的玩家确认界面，如果用多线程模拟的话，那么它应该是下面的样子：

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/27/16247872044353.jpg)

主线程创建了5个子线程，各子任务执行确认动作，**期间主线程进入等待状态，直到各子线程的任务均已经完成，主线程恢复继续执行**，也就是游戏继续。而如果其中某个玩家超时未执行确认的话，那么主线程将结束本次匹配，重新开始新一轮的匹配。

这个场景，就是CountDownLatch适用的第一个经典场景。

**场景2. 协调子线程开始动作：统一各线程动作开始的时机**

这个场景的例子也十分常见。比如，田径场上，各选手各就各位等待发令枪。在发令枪响之前，选手只能原地就位，否则就是违规。如果从多线程的角度看，这恰似你创建了一些多线程，但是你需要统一管理它们的任务开始时间。**因为，如果你不对此做干预的话，线程调用`start(）`之后的具体时间是不确定的，这个知识点我们早在青铜系列文章中就已经讲过**。

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/27/16247889354820.jpg)

在王者中也有类似的场景，游戏开始时，各玩家的初始状态必须一致。**总不能，你刚降生，对方已经打到你家门口了**。


![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/27/16248078166227.jpg)

上述的两个场景的问题，正是CountDownLatch所要解决的问题。理解了这两个问题，你也就理解了CountDownLatch存在的价值。

## 二、Java中的CountDownLatch设计

JUC中CountDownLatch的实现，是以类的形式存在，而不是接口，你可以直接拿过来使用。并且，它的实现还很简单。在数据结构上，CountDownLatch基于一个同步器实现，你可以看它的`final Sync sync`变量。

而在构造函数上，CountDownLatch有且只有`CountDownLatch(int count)`一个构造器，并且你需要指定数量，**并且你不得在中途修改它，这点务必牢记**！

**核心函数**

* `await()`：等待latch降为0；
* `boolean await(long timeout, TimeUnit unit)`：等待latch降为0，但是可以设置超时时间。比如有玩家超时未确认，那就重新匹配，总不能为了某个玩家等到天荒地老吧。
* `countDown()`：latch数量减1；
* `getCount()`：获取当前的latch数量。

从CountDownLatch的方法上看，还是比较简单易懂的，重点要理解`await()`和`countDown()`.

## 三、CountDownLatch如何解决场景问题

接下来，我们将第一小节的两个场景问题用代码实现一遍，让你对CountDownLatch的用法有个直观的理解。

**场景1. CountDownLatch实现对各子线程的等待**

创建大乔、兰陵王、安其拉、哪吒和铠等五个玩家，主线程必须在他们都完成确认后，才可以继续运行。

在这段代码中，`new CountDownLatch(5)`用户创建初始的latch数量，各玩家通过`countDownLatch.countDown()`完成状态确认。

```java
public static void main(String[] args) throws InterruptedException {
    CountDownLatch countDownLatch = new CountDownLatch(5);

    Thread 大乔 = new Thread(countDownLatch::countDown);
    Thread 兰陵王 = new Thread(countDownLatch::countDown);
    Thread 安其拉 = new Thread(countDownLatch::countDown);
    Thread 哪吒 = new Thread(countDownLatch::countDown);
    Thread 铠 = new Thread(() -> {
        try {
            // 稍等，上个卫生间，马上到...
            Thread.sleep(1500);
            countDownLatch.countDown();
        } catch (InterruptedException ignored) {}
    });

    大乔.start();
    兰陵王.start();
    安其拉.start();
    哪吒.start();
    铠.start();
    countDownLatch.await();
    System.out.println("所有玩家已经就位！");
}
```

**场景2. CountDownLatch实现对多线程的统一管理**

在这个场景中，我们仍然用五个线程代表大乔、兰陵王、安其拉、哪吒和铠等五个玩家。**需要注意的是，各玩家虽然都调用了`start()`线程，但是它们在运行时都在等待countDownLatch的信号，在信号未收到前，它们不会往下执行**。

```java
public static void main(String[] args) throws InterruptedException {
    CountDownLatch countDownLatch = new CountDownLatch(1);

    Thread 大乔 = new Thread(() -> waitToFight(countDownLatch));
    Thread 兰陵王 = new Thread(() -> waitToFight(countDownLatch));
    Thread 安其拉 = new Thread(() -> waitToFight(countDownLatch));
    Thread 哪吒 = new Thread(() -> waitToFight(countDownLatch));
    Thread 铠 = new Thread(() -> waitToFight(countDownLatch));

    大乔.start();
    兰陵王.start();
    安其拉.start();
    哪吒.start();
    铠.start();
    Thread.sleep(1000);
    countDownLatch.countDown();
    System.out.println("敌方还有5秒达到战场，全军出击！");
}

private static void waitToFight(CountDownLatch countDownLatch) {
    try {
        countDownLatch.await(); // 在此等待信号再继续
        System.out.println("收到，发起进攻！");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

运行结果如下，各玩家在收到出击信号发起了进攻：

```shell
敌方还有5秒达到战场，全军出击！
收到，发起进攻！
收到，发起进攻！
收到，发起进攻！
收到，发起进攻！
收到，发起进攻！

Process finished with exit code 0
```

## 小结

以上就是关于CountDownLatch的全部内容。总体上，CountDownLatch比较简单且易于理解。在学习时，先了解其设计意图，再写个Demo基本就能流畅掌握。

正文到此结束，恭喜你又上了一颗星✨

## 夫子的试炼

* 编写代码体验CountDownLatch用法。

## 延伸阅读与参考资料

* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。

