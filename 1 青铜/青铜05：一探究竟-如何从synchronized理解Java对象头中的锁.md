欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第5篇**。

在前面的文章《青铜4：synchronized用法初体验》中，我们已经提到**锁**的概念，并指出`synchronized`是锁机制的一种实现。可是，这么说未免太过抽象，你可能无法直观地理解**锁究竟是什么**？所以，本文会粗略地介绍`synchronized`背后的一些基本原理，让你对Java中的锁有个粗略但直观的印象。

本文将分两个部分，首先你要从Mark Word中认识锁，因为对象锁的信息存在于Mark Word中，其次通过JOL工具实际体验Mark Word的变化。

## 一、从Mark Word认识锁

我们知道，在HotSpot虚拟机中，一个对象的存储分布由3个部分组成：
* **对象头（Header）**：由**Mark Word**和**Klass Pointer**组成；
* **实例数据（Instance Data）**：对象的成员变量及数据；
* **对齐填充（Padding）**：对齐填充的字节，暂时不必理会。

在这3个部分中，对象头中的**Mark Word**是本文的重点，也是理解Java锁的关键。Mark Word记录的是对象运行时的数据，其中包括：
* **哈希码（identity_hashcode）**
* **GC分代年龄（age）**
* **锁状态标志**
* **线程持有的锁**
* **偏向线程ID（thread）**

所以，从对象头中的Mark Word看，**Java中的锁就是对象头中的一种数据**。在JVM中，**每个对象都有这样的锁**，并且用于多线程访问对象时的并发控制。

如果一个线程想访问某个对象的实例，那么这个线程必须拥有该对象的**锁**。首先，它需要通过对象头中的Mark Word判断该对象的实例是否已经被线程锁定。如果没有锁定，**那么线程会在Mark Word中写入一些标记数据**，就是告诉别人：这个对象是我的啦！如果其他线程想访问这个实例的话，就需要进入等待队列，直到当前的线程释放对象的锁，也就是把Mark Word中的数据擦除。

**当一个线程拥有了锁之后，它便可以多次进入**。当然，在这个线程释放锁的时候，那么也需要执行相同次数的释放动作。比如，一个线程先后3次获得了锁，那么它也需要释放3次，其他线程才可以继续访问。

下面的表格展示的是64位计算机中的对象头信息：
```
|------------------------------------------------------------------------------------------------------------|--------------------|
|                                            Object Header (128 bits)                                        |        State       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                  Mark Word (64 bits)                         |    Klass Word (64 bits)     |                    |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Normal       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| thread:54 |       epoch:2        | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Biased       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                       ptr_to_lock_record:62                         | lock:2 |    OOP to metadata object   | Lightweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                     ptr_to_heavyweight_monitor:62                   | lock:2 |    OOP to metadata object   | Heavyweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                                                     | lock:2 |    OOP to metadata object   |    Marked for GC   |
|------------------------------------------------------------------------------|-----------------------------|--------------------|

```

从表格中，你可以看到Object Header中的三部分信息：Mark Word、Klass Word、State.

## 二、通过JOL体验Mark Word的变化
为了直观感受对象头中Mark Word的变化，我们可以通过 **JOL（Java Object Layout）** 工具演示一遍。JOL是一个不错的Java内存布局查看工具，希望你能记住它。

首先，在工程中引入依赖：

```
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.10</version>
</dependency>
```
在下面的代码中，`master`是我们创建的对象实例，方法`decreaseBlood()`中会执行加锁动作。所以，在调用`decreaseBlood()`加锁后，**对象头信息应该会发生变化**。
```java
 public static void main(String[] args) {
        Master master = new Master();
        System.out.println("====加锁前====");
        System.out.println(ClassLayout.parseInstance(master).toPrintable());
        System.out.println("====加锁后====");
        synchronized (master) {
            System.out.println(ClassLayout.parseInstance(master).toPrintable());
        }
    }
```

结果输出如下：
```
====加锁前====
cn.tao.king.juc.execises1.Master object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    int Master.blood                              100
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

====加锁后====
cn.tao.king.juc.execises1.Master object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           48 f9 d6 00 (01001000 11111001 11010110 00000000) (14088520)
      4     4        (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
      8     4        (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4    int Master.blood                              95
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total


Process finished with exit code 0
```

从结果中可以看到，代码在执行`synchronized`方法后，所打印出的`object header`信息由`01 00 00 00`、`00 00 00 00`变成了`48 f9 d6 00`、`00 70 00 00`等等，不出意外的话，相信你应该看不明白这些内容的含义。

所以，为了方便阅读，我们在青铜系列文章《**借花献佛-JOL格式化工具**》中提供了一个工具类，让输出更具可读性。借助工具类，我们把代码调整为：

```java
 public static void main(String[] args) {
        Master master = new Master();
        System.out.println("====加锁前====");
        printObjectHeader(master);
        System.out.println("====加锁后====");
        synchronized (master) {
            printObjectHeader(master);
        }
    }
```

输出的结果如下：

```
====加锁前====
# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
Class Pointer: 11111000 00000000 11000001 01000011 
Mark Word:
	hashcode (31bit): 0000000 00000000 00000000 00000000 
	age (4bit): 0000
	biasedLockFlag (1bit): 0
	LockFlag (2bit): 01

====加锁后====
Class Pointer: 11111000 00000000 11000001 01000011 
Mark Word:
	javaThread*(62bit,include zero padding): 00000000 00000000 01110000 00000000 00000100 11100100 11101001 100100
	LockFlag (2bit): 00
```

你看，这样一来，输出的结果的结果就一目了然。从**加锁后**的结果中可以看到，Mark Word已经发生变化，当前线程已经获得对象的锁。

**至此，你应该明白，原来synchronized的背后的原理是这么回事**。当然，本文所讲述只是其中的部分。出于篇幅考虑和难度控制，**本文暂且不会对Java对象头中锁的含义和锁的升级等问题展开描述**，这部分内容会在后面的文章中详细介绍。

以上就是文本的全部内容，恭喜你又上了一颗星✨

## 夫子的试炼

* 下载JOL工具，在代码中体验工具的使用和对象信息的变化。

## 延伸阅读与参考资料

* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。