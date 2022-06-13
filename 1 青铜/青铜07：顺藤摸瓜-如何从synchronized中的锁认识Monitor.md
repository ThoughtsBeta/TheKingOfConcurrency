欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第7篇**。

在前面的文章中，我们已经体验过synchronized的用法，并对锁的概念和原理做了简单的介绍。然而，你可能已经察觉到，**有一个概念**似乎总是和**synchronized**、**锁**这两个概念**如影相随**，很多人也比较喜欢问它们之间的区别，这个概念就是**Monitor**，也叫**监视器**。

所以，在讲解完synchronized、锁之后，文本将为你讲解Monitor，揭示它们之间那些公开的秘密，希望你不再迷惑。


首先，你要明白的是，**Monitor作为一种同步机制**，它并非Java所特有，但Java实现了这一机制。

为了具象地理解Monitor这一抽象概念，我们先来分析身边的一个常见场景。

## 一、从医院排队就诊机制理解Monitor

相信你一定有过去医院就诊的经历。我们去医院时，情况一般是这样的：

* 首先，我们在**门诊大厅**前台或自助挂号机**进行挂号**；
* 随后，挂号结束后我们找到对应的**诊室就诊**：
    * 诊室每次只能有一个患者就诊；
    * 如果此时诊室空闲，直接进入就诊；
    * 如果此时诊室内有患者正在就诊，那么我们进入**候诊室**，等待叫号；
* 就诊结束后，**走出就诊室**，候诊室的**下一位候诊患者**进入就诊室。

这个就诊过程你一定耳熟能详，理解起来必然毫不费力气。我们做了一张图展示图下：

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/05/30/16223788362085.jpg)

仔细看这幅图中的就诊过程，**如果你理解了这个过程，你就理解了Monitor**. 这么简单吗？不要怀疑自己，是的。**你竟然早已理解Monitor机制**！

不要小看这个机制，它可是生活中的智慧体现。在这个就诊机制中，它起到了**两个关键性的作用**：
* **互斥（mutual exclusion ）**：每次只允许一个患者进入候诊室就诊；
* **协作（cooperation）**：就诊室中的患者就诊结束后，可以通知候诊区的下一位患者。

明白了吗？你在医院就诊的过程竟然和Monitor的机制几乎一模一样。我们换个方式来描述Monitor在计算机科学中的作用：

* **互斥（mutual exclusion ）**：每次只允许一个线程进入临界区；
* **协作（cooperation）**：当临界区的线程执行结束后满足特定条件时，可以通知其他的等待线程进入。

而就诊过程中的**门诊大厅**、**就诊室**、**候诊室**则恰好对应着Monitor中的三个关键概念。其中：

* **门诊大厅**：所有待进入的线程都必须先在**入口（Entry Set）**挂号才有资格；
* **就诊室**：一个每次只能有一个线程进入的**特殊房间（Special Room）**；
* **候诊室**：就诊室繁忙时，进入**等待区（Wait Set）**；

我们把上面的图稍作调整，就可以看到Monitor在Java中的模样：
![](https://writting.oss-cn-beijing.aliyuncs.com/2022/06/10/16548665049934.jpg)

对比来看，相信你已经很直观地理解Monitor机制。再一回味，你会发现**synchronized正是对Monitor机制的一种实现**。而在Java中，**每一个对象都会关联一个监视器**。

## 二、从synchronized源码感受Monitor

既然synchronized是对Monitor机制的一种实现，为了让你更有体感，我们可以写一段**极简代码**一探究竟。

这段代码极为简单，但是够用，我们在代码中使用了`synchronized`关键字：

```java
public class SyncMonitorDemo {
    public static void main(String[] args) {
        Object o = new Object();
        synchronized (o) {
            System.out.println("locking...");
        }
    }
}
```
代码写好后，分别执行`javac SyncMonitorDemo.java`和 `javap -v SyncMonitorDemo.class`，随后你就能得到下面这样的字节码：

```
Classfile SyncMonitorDemo.class
  Last modified May 26, 2021; size 684 bytes
  MD5 checksum e366920f22845e98c45f26531596d6cf
  Compiled from "SyncMonitorDemo.java"
public class cn.tao.king.juc.execises1.SyncMonitorDemo
  minor version: 0
  major version: 49
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #2.#22         // java/lang/Object."<init>":()V
   #2 = Class              #23            // java/lang/Object
   #3 = Fieldref           #24.#25        // java/lang/System.out:Ljava/io/PrintStream;
   #4 = String             #26            // locking...
   #5 = Methodref          #27.#28        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #6 = Class              #29            // cn/tao/king/juc/execises1/SyncMonitorDemo
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcn/tao/king/juc/execises1/SyncMonitorDemo;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               o
  #19 = Utf8               Ljava/lang/Object;
  #20 = Utf8               SourceFile
  #21 = Utf8               SyncMonitorDemo.java
  #22 = NameAndType        #7:#8          // "<init>":()V
  #23 = Utf8               java/lang/Object
  #24 = Class              #30            // java/lang/System
  #25 = NameAndType        #31:#32        // out:Ljava/io/PrintStream;
  #26 = Utf8               locking...
  #27 = Class              #33            // java/io/PrintStream
  #28 = NameAndType        #34:#35        // println:(Ljava/lang/String;)V
  #29 = Utf8               cn/tao/king/juc/execises1/SyncMonitorDemo
  #30 = Utf8               java/lang/System
  #31 = Utf8               out
  #32 = Utf8               Ljava/io/PrintStream;
  #33 = Utf8               java/io/PrintStream
  #34 = Utf8               println
  #35 = Utf8               (Ljava/lang/String;)V
{
  public cn.tao.king.juc.execises1.SyncMonitorDemo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcn/tao/king/juc/execises1/SyncMonitorDemo;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: new           #2                  // class java/lang/Object
         3: dup
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: astore_1
         8: aload_1
         9: dup
        10: astore_2
        11: monitorenter
        12: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
        15: ldc           #4                  // String locking...
        17: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        20: aload_2
        21: monitorexit
        22: goto          30
        25: astore_3
        26: aload_2
        27: monitorexit
        28: aload_3
        29: athrow
        30: return
      Exception table:
         from    to  target type
            12    22    25   any
            25    28    25   any
      LineNumberTable:
        line 5: 0
        line 6: 8
        line 7: 12
        line 8: 20
        line 9: 30
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      31     0  args   [Ljava/lang/String;
            8      23     1     o   Ljava/lang/Object;
}
SourceFile: "SyncMonitorDemo.java"
```

`javap`是JDK自带的一个反汇编命令。你可以忽略其他不必要的信息，直接在结果中找到下面这段代码：

```
11: monitorenter
12: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
15: ldc           #4                  // String locking...
17: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
20: aload_2
21: monitorexit
```

看到`monitorenter`和`monitorexit`指令，相信智慧的你已经看穿一切。

以上就是文本的全部内容，恭喜你又上了一颗星✨

## 夫子的试炼

* 写一段包含`synchronized`关键字的代码，使用`javap`命令观察结果。

## 延伸阅读与参考资料

* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。