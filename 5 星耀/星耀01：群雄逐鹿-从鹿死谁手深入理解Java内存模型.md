欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第27篇**，星耀中的**第1篇**。

在前面青铜、黄金、铂金和砖石系列文章中，我们已经介绍了JAVA并发编程中的常见问题和基本的解决思路。然而，这些文章更多的是侧重于如何理解并使用JAVA既有的工具，并没有谈及工具的原理。所以，在星耀系列中，我们的主题将会侧重于JAVA并发原理的介绍，比如JAVA并发模型、AQS的设计原理等。

在本篇文章中，我们将首先介绍JAVA内存模型。提到JAVA内存模型，我们可能会首先想到那幅经典的内存模型图，不怎么好看但似乎还挺复杂，让人有种莫名的望而却步的感觉，如果再扯上本文要讲的**指令重排**、**Happens-before**、**volatile**和**synchronized**，似乎难上加难。但其实，我们可以换个思路来理解JAVA内存模型，或许能轻松点，不妨试着往下看。

## 一、群雄逐鹿，鹿死谁手？

这一切，还得从王者峡谷中跌宕起伏的纷争说起。

月黑风高夜，铠爷拖着利剑在峡谷游荡，山谷里的生活毕竟枯燥且乏味，夜晚打野算是消遣。溪水潺潺处，一只小鹿在悠闲地喝水。要说这小鹿也是倒霉，它哪里知道喝口水竟然遇见了打野， 它背后的男人正挥舞着利剑向它走来，已注定在劫难逃。然而，就在铠即将斩获这只小鹿时，好巧不巧，他的老对手兰陵王从草丛里蹦了出来，真是冤家路窄，哪哪都有这人。于是，这俩打野痴汉开启了对小鹿的你争我夺。	

那么，问题来了。按照峡谷的规矩，这鹿最终定然不可能同属于二人，谁补了最后一刀，这鹿就算是谁的。可是，如何界定是谁补的最后一刀？

![image-20220207121437102](https://writting.oss-cn-beijing.aliyuncs.com/image-20220207121437102.png)

这个问题的背后，就是个典型的并发读写问题。接下来，我们写段程序来感受下这个血腥的场面，并试着找出解决的办法。

我们定义一个`DeerGame`类来模拟这次的竞争。在这个场景中，有一只待宰的小鹿，它有**100**个单位的血量，每次被攻击时都掉部分的血量，血量为**0**时，小鹿将丢掉性命，攻击者获胜。

```java
/**
 * 群雄逐鹿
 */
public class DeerGame {
   /**
    * 待宰的小鹿
    */
    private final Deer deer = new Deer();

    /**
     * 物理攻击，一次攻击掉血10个单位
     */
    public boolean physicalAttack() {
        return deer.reduceBlood(10) == 0;
    }

    /**
     * 魔法攻击，一次攻击掉血5个单位
     */
    public boolean magicAttack() {
        return deer.reduceBlood(5) == 0;
    }

    @Data
    private static class Deer {
        private int blood = 100;

        public int reduceBlood(int bloodToReduce) {
            int remainBlood = blood - bloodToReduce;
            blood = Math.max(remainBlood, 0);
            return blood;
        }
    }
}
```

接着，我们创建两个线程来模型**兰陵王**和**铠**对小鹿的争夺，他们将各自对小鹿进行多达**100次**的物理攻击（有点毫无人性）。谁是最后一击，谁就是获胜的一方。

```java
public static void main(String[] args) {
    DeerGame deerGame = new DeerGame();
    Thread 兰陵王 = new Thread(() -> {
        for (int i = 0; i < 100; i++) {
            if (deerGame.physicalAttack()) {
                System.out.println("兰陵王胜出！");
            }
        }
    });

    Thread 铠 = new Thread(() -> {
        for (int i = 0; i < 100; i++) {
            if (deerGame.physicalAttack()) {
                System.out.println("铠胜出！");
            }
        }
    });
    兰陵王.start();
    铠.start();
}
```

看到这里，如果你阅读过前面的系列文章，或者有一定的并发编程基础，那么你一定会发现上述代码片段存在严重的缺陷：**它没有并发控制**。换句话说，**两个线程都在读写Deer中的`blood`字段，但这个字段却没有任何的并发处理**，结果就是程序故障，两人可能都获胜。也许你会说，解决这个问题很简单，我们可以在`reduceBlood`方法前面加上`synchronized`关键字来实现多线程同步。

**你说的很对，但我们要讨论的问题正在于此**。为什么要加上`synchronized`关键字？**加与不加，内存层面发生了什么事？** 虽然解决的办法很简单，但要理解这个办法的内涵却并不容易，这个过程将会涉及到JAVA内存的并发模型设计、硬件架构设计、CPU指令重排等多个问题，请稍作镇定继续往下看，我们一起来啃下这块硬骨头。

## 二、内存世界里的软硬件架构差异

### （一）从软件层面理解JAVA内存模型设计

在上述的代码片段中，我们定义了`Deer`对象，在Deer中有个`blood`字段。此外，还有个`reduceBlood`方法，注意在这个方法中的	`remainBlood`变量。

```java

public class DeerGame {
    private final Deer deer = new Deer();
    @Data
    private static class Deer {
        private int blood = 100;

        public int reduceBlood(int bloodToReduce) {
            int remainBlood = blood - bloodToReduce;
            blood = Math.max(remainBlood, 0);
            return blood;
        }
    }
}
```

在这小小的代码片段中，已经包含了线程的**堆栈**、**本地变量**和**共享对象**等内容。所以，我们现在有必要回顾下JAVA的内存模型设计。当然，我们在此不会展开描述，如果你需要了解更多，可以参考[这篇文章](https://juejin.cn/post/6981790758290358302)。**在此，我们要理解的是，从JAVA模型角度来说，对象与线程的数据放哪里了**。

在JAVA虚拟机中，内存主要分为**堆（Heap）**与**栈（Stack）**两个部分。其中，堆存储的是共享数据，而栈存储的则是线程的专属数据，每个线程都有自己的**线程栈（thread stack）**。线程栈包含了当前线程所访问的方法以及当前的访问点，所以线程栈也可以称之为**调用栈（call stack）**。 此外，线程栈还包含了所执行方法的本地变量，比如上述示例代码中的` remainBlood`，线程中的本地变量存储在线程栈中，所以它们仅对当前线程可见，其他线程想看是不可能的，若想访问那更是痴心妄想。常见的 原始类型数据都会存储在线程栈中，比如`boolean`、`byte`、`short`、 `char`、 `int`、 `long`、 `float`和`double`等。原始变量在传递时，传递是的是拷贝的副本，并不会传递本身。

![image-20220207170302879](https://writting.oss-cn-beijing.aliyuncs.com/image-20220207170302879.png)

到这里，我们理解每个线程都有自己的小房间来存储自己的数据，在传递原始类型时，传递的也只是副本，看起来安全且祥和。**但是，假如线程间传递的是引用类型呢**？比如，我们传递的不是`int`和`boolean`，而是`Integer`和`Boolean.`..情况会怎么样？又比如，我们把上述示例代码调整，两个线程中分别有自己的本地变量`deerGame1`和`deerGame2`，但是这两个变量的引用都是`deerGame`，所以它们两个注定要纠缠不清。

```java
private static final DeerGame deerGame = new DeerGame();

public static void main(String[] args) {
    Thread 兰陵王 = new Thread(() -> {
        DeerGame deerGame1 = deerGame;
        for (int i = 0; i < 100; i++) {
            if (deerGame1.physicalAttack()) {
                System.out.println("兰陵王胜出！");
            }
        }
    });
    Thread 铠 = new Thread(() -> {
        DeerGame deerGame2 = deerGame;
        for (int i = 0; i < 100; i++) {
            if (deerGame2.physicalAttack()) {
                System.out.println("铠胜出！");
            }
        }
    });
    兰陵王.start();
    铠.start();
}
```

从内存模型理解的话，就是本地变量`deerGame1`和`deerGame2`都引用了`deerGame`，虽然本地变量存储在线程栈中，但是`deerGame`却位于堆上，属于共享数据。**对于共享数据，无论是谁修改了它，如果引用它的线程数据未能及时感知，那么注定是一场悲剧**。

![](https://writting.oss-cn-beijing.aliyuncs.com/image-20220207170320238.png)

**那么，为什么共享的数据在变化时不能被所有的线程很好的感知呢？** 这是个好问题。接下来，我们就要从硬件架构方面来理解这个问题为什么会存在。

### （二）从硬件层面理解内存架构

从硬件层面来说，现代计算机的内存架构可以用下面这幅图来简要说明，包含了**主内存**、**CPU高速缓存**和**CPU寄存器**等。从数据读取的速度上来说，很明显离CPU越近其速度越快，所以最快的是CPU寄存器，其次是CPU高速缓存，最后是主存。

![image-20220207115909930](https://writting.oss-cn-beijing.aliyuncs.com/image-20220207115909930.png)



**凡是涉及缓存的地方，就必然涉及数据一致性的问题。所以，理解硬件层面的架构最重要的是要能理解多级缓存的存在**。正是因为多级缓存的存在，CPU的运算才变得复杂和多变。

所有的CPU在运算时，都能访问寄存器、高速缓存和主存。**通常，当 CPU 需要访问主存时，它会将部分主存读入其 CPU 缓存。它甚至可以将部分缓存读入其内部寄存器，然后对其执行操作。当 CPU 需要将结果写回主存时，它会将值从其内部寄存器刷新到高速缓存，并在某个时候将值刷新回主存**。而当 CPU 需要在高速缓存中存储其他内容时，通常会将存储在高速缓存中的值刷新回主内存。

看到这里，可能你已经产生了疑问：**不同的线程运行在不同的CPU中，每个CPU又有自己的缓存，那线程之间如何共享数据？** 好问题，这正是我们接下来要讨论的。

### （三）理解软硬件的架构差异

从上述软硬件两部分的内容，我们可以看到，JVM的内存模型设计与硬件的缓存体系设计是不同的。在JVM中，我们将存储分为堆和栈两个部分，然而在硬件内存的设计却没有这种划分，这就导致了下图所示的问题。JVM中大部分的堆栈数据都存储在主存，但是有部分线程的数据存储在寄存器或高速缓存中。**如此，麻烦就来了：不同线程对共享变量的读与写怎么搞**？我更新的变量别人怎么看得见？别人把对象更新了我还怎么用？所以，这就产生了**共享对象可见性**和**竞态条件**两个关键问题。

![](https://writting.oss-cn-beijing.aliyuncs.com/image-20220207170357175.png)

#### 1. 共享对象的可见性

前面我们已经讲过不同线程都有自己的空间来存储自己的数据，不同的CPU也有自己的寄存器和高速缓存来存储数据。对于单个线程来说，线程更新了主存中的变量，在没有其他线程更改的情况下，它总是能读取到之前所更新的值，不会有任何问题。但是，对于多线程来说，这就会产生所谓的 **共享对象的可见性（Visibility of Shared Objects）** 问题。

比如，在上述示例代码中，铠从主存中读取的`blood`的值为2，随后将值改成了1，在他还未将1刷回主存的时候，兰陵王也从存主存中读取到了2. 那么，这个时候数据的不一致性就产生了，兰陵王无法看到铠对数据的修改，很显然他所拿到的**数据已经失效**。

当然，解决可见性的简单而有效的办法就是使用`volatile`关键字。**`volatile`可以确保不同线程在读取变量时，总是从主存读取，在更新缓存时也更新到主存**。关于`volatile`的更多内容，我们稍后再讲。

![image-20220208102609630](https://writting.oss-cn-beijing.aliyuncs.com/image-20220208102609630.png)

#### 2. 竞态条件

现在，我们已经理解所谓共享对象的可见性是指线程修改数据后，其他线程能否感知看见的问题。那么，如果修改共享对象的不是某个线程，而是多个线程同时修改会怎么样？这就产生了另外一个问题，即**竞态条件（Race Conditions）**。

如下图所示，铠和兰陵王同时从主存中获取了`blood=2`，随后他们在各自的缓存中将`blood`的值更改为1，并相继将更新后的值刷回主存。**但是，此时无论谁先执行刷回主存的动作，主存最后的结果都是1**。很显然，我们期望的结果应该是0，数据已经出错了，这就是竞态条件下的并发问题。

其实，要解决竞态条件的问题并不难，我们自然能想到使用同步，比如`synchronized`可以让同一时刻仅有一个线程可以访问临界区域。同时，线程在同步块内读取变量时，都会从主存中读取，而在线程退出同步块时，则会将变量刷回主存。

![image-20220208102521978](https://writting.oss-cn-beijing.aliyuncs.com/image-20220208102521978.png)

你看，可见性和竞态条件这两个问题都不是凭空而来，而是由问题催生了这两个问题。所以，下面的内容就是围绕着两个问题来寻求答案。

## 三、调和软硬件架构差异

在上文我们讲述了JAVA内存模型的设计和硬件内存的架构，以及它们之间的差异所导致的问题。那如何调和它们之间的差异？这就需要分别从CPU和JAVA模型设计上找到一些答案，也就是我们要谈论的**指令重排**、**Happens-before**、**volatile**和 **synchronized**等概念。

**但是，为什么是这几个概念而是不是其他？**

说到这里，我们要理解的是JAVA内存模型中最重要的就是对**各种操作**的处理。要编排好这些操作，就需要按照**Happens-before**偏序关系对它们进行排序，而这种操作是基于内存操作和同步操作等级别来定义的。如果缺少充足的同步，当不同的线程访问共享数据时，就会发生很多奇怪的问题，所以就需要**volatile**和 **synchronized**等措施来规避这些问题。

你看，原本零散的概念是不是就衔接起来了？接下来，我们试着给它们多点描述。

### （一)理解Happens-before关系

#### 1. 理解CPU下的指令重排

首先不得不提的就是**指令重排（Instruction Reordering）**，你可能在其他地方也看见过这个概念。**为什么要先说指令重排？因为顺序决定着CPU的运算结果**。无论是共享对象的可见性问题，还是竞态条件问题，**本质上都是顺序的问题**。并且，正是因为顺序的问题，才有了后面的Happens-before关系问题。

从设计初衷上讲，**指令重排是为了提高CPU的处理性能，让CPU并发运算地更快**。比如，10个人去野外打野烧烤，如果这10人总是一起打野、一起捡祡火、一起烤制，那就不如分组后同时分头行动更有效率，打野的回来时，柴火也捡好了。开干，很麻溜。CPU的指令重排要做的就是按照合理的工序，给这10个人分组，然后分头行动。**当然，分工的时候，要考虑到上下游的依赖问题。如果调度器采用不恰当的方式来交替执行不同线程的操作，将会导致不正确的结果**。

看下面这两个简单的语句，语句1和语句2之间没有任何的依赖的依赖关系。如此，CPU在运算时可以并发地执行这两个语句，这样要比逐条执行快很多。

```java
statement1: a = a1 + a2
statement2: b = b1 + b2
```
但是，如果我们把上面两个语句调整为下面这样，两个语句不再是独立的语句，语句2依赖于语句1先执行，此时如果并发执行会发生什么？很显然，如果不加以控制，这两条语句的执行顺序可能很随机，每次执行的结果都不一样。于是，问题就来了。

```java
statement1: a = a1 + a2
statement2: b = a + b1
```
CPU在执行程序指令时，为了提高处理的效率，并不会傻傻地按顺序执行。CPU会对需要执行的指令进行分析、调整执行的顺序，使得指令集既能并发执行也不会影响最终的结果。

#### 2. 指令重排下的Happens-before关系约定

从静态的视角看，JAVA内存模型可以分为堆栈两大块。然而，如果从动态的视角看，JAVA内存模型其实是通过**各种操作**来定义的，比如对变量的读/写操作、对监视器的加锁/释放操作，以及线程的启动/协调等操作。**为了正确编排、执行这些操作，就需要在它们之间建立一种偏序关系，也就是Happens-before**。

抽象地讲，所谓Happens-before指的是两个**事件的结果**之间的关系。**如果一个事件应该在另外一个事件之前发生，则结果必须反映这一点**。举个例子，我们排序接种新冠疫苗，接种点制定的规则是谁先到谁先接种。那么，如果我比你先到，我自然就一定要在你前面接种。

当铠和兰陵王同时猎杀小鹿时，如果是铠先动的手，那么在程序执行的结果上一定是铠获胜，这和兰陵王后面蹦出来跳得多高没有关系。**所以，这是约定的基本原则，也就是Happens-before关系。如果线程之间建立了这样的关系，那么不同线程的操作应当被其他线程所看见，否则就会出现内存一致性错误（Memory Consistency Errors）**。

至于如何建立这样的关系，那就是我们程序设计上要考虑的事情了。

![image-20220208144656329](https://writting.oss-cn-beijing.aliyuncs.com/image-20220208144656329.png)

### （二）如何建立Happens-before关系

通常，我们有几种方式来建立Happens-before关系，比如使用单线程、Thread中的start/join等，**但是最重要的两种方式还是`synchronized`和 `volatile`**，有两句英文对此有很好的概括：

- “A volatile write will happen before another volatile read。”

- “An unlock on the synchronized block will happen before another lock。”

#### 1. 使用synchronized同步块

当不同的线程访问临界区时，使用`synchronized`关键字是个简单有效的方案。在同一时刻，只有获得锁的线程才能执行临界区的代码块，而未获得锁的线程自然只能在后面等待。**所以，获得锁的线程和其他未获得锁的线程之间，就具有Happens-before关系**。

#### 2. 使用volatile关键字

为了解决指令重新排序的问题，Java提供了 `volatile`关键字，它不仅可以保证共享对象的可见性，还提供了Happens-before关系保证：

* **变量写入的可见性保证**：当写入 `volatile`变量时，该值保证会直接写入主内存 (RAM)。此外，**写入`volatile` 变量的线程可见的所有变量也将同步到主存储器**；

* **变量读取的可见性保证**：当读取 `volatile`的值时，可以保证直接从内存中读取该值。此外，**读取 volatile 变量的线程可见的所有变量也将从主存储器中刷新它们的值。**

**volatile关键字的局限性**

虽然`volatile`关键字保证对`volatile`变量的所有读取都直接从主存读取，并且对`volatile`变量的所有写入都直接写入主存，但应对并发场景时，单单依靠`volatile`仍然是不够的。比如，前面所说的多个线程同时更新某个变量时，就会发生竞态条件，彼此会覆盖他人已更新的值。

**什么情况下使用volatile是可靠的**

如果两个线程都在读取和写入共享变量，那么使用 `volatile`关键字是不够的。在这种情况下，我们需要使用**同步**来保证变量的读取和写入是原子的，所以`volatile`往往和`synchronized`或其他JUC中的并发工具搭配使用。

虽然多线程读写的场景下使用`volatile`并不可靠，但是如果只有一个线程读写，而其他线程只读的话，那么将会保证其他线程所读取的都是最新值。如果我们不使用`volatile`的话，则无法做到这点。

**volatile的性能考量**

由于`volatile`变量的读写都是发生在主存，那么很显然CPU对此类变量的读取会影响到性能。毕竟，`volatile`变量无法再享受CPU寄存器和高速缓存的性能优势。所以，我们在使用`volatile`时需要考虑到这点。


## 小结

至此，关于JAVA内存并发模型的介绍到此结束。在本文中，我们通过铠和兰陵王争夺小鹿的例子来引出内存中的并发问题。接着，我们通过对硬件架构的描述来进一步说明内存模型中的数据一致性问题的产生及原因。最后，根据问题我们去寻找答案，并陆续提到了指令重排、Happens-before、volatile和synchronized等概念及作用。

本文的内存较为枯燥不易理解，涉及到了软硬件的架构和很多的概念。在学习时，建议不要把它们看作是独立的知识点，这样理解起来会比较生硬，而是要把它们和要解决的问题联系起来，完成知识的串联，形成整体的结构化认知。**换句话说，我们要理解它是什么，更要理解为什么是它**。

比如，我们通过小鹿的例子感知到了内存模型中的问题，从问题我们又追溯到硬件架构的设计，至此应理解软硬件架构的差异是问题的所在。那如何调和差异呢？于是我们又从硬件架构的角度去理解指令重排和Happens-before的概念，为了约束指令重排和构建Happens-before，volatile和synchronized闪亮登场。这样，我们就可以把本文的很多概念串联起来。

正文到此结束，恭喜你又上了一颗星✨

## 夫子的试炼

* 动手：编写多线程代码体验`volatile`变量和非`volatile`变量读写的不同效果。

## 延伸阅读与参考资料

* 《王者并发课》大纲与更新进度总览：https://juejin.cn/post/6967277362455150628
* https://www.baeldung.com/java-volatile
* https://www.logicbig.com/tutorials/core-java-tutorial/java-multi-threading/happens-before.html
* http://tutorials.jenkov.com/java-concurrency/java-memory-model.html
* https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.5
* 掘金专栏：https://juejin.cn/column/6963590682602635294
* Github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。