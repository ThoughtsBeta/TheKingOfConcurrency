欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第28篇**，星耀中的**第2篇**。

说起JUC中大名鼎鼎的**AQS（AbstractQueuedSynchronizer）**，相信大家对其并不陌生，说是如雷贯耳也不为过。然而，当我们需要走近它的时候，大部分人或望而却步，或绕道而行。原因在于，虽然我们有了解它的欲望，但想把它搞清楚又似乎总是不容易。**正所谓撼山易，撼AQS难**。

作为星耀系列的文章，解决的正是此类难解之题。所以，我们将在这篇文章中邀你一起，用庖丁解牛的方式来逐步剖析AQS的表象与内核，理解AQS的原理及应用。在本文中，我们不会开篇就直捣黄龙、解读源码，而是**从最基本的同步说起，并由同步循序渐进地讲解同步器、CLH队列和AQS的基本构成，以及AQS在ReentrantLock和CountDownLatch中的应用，最终实现解构AQS的目的**。

## 一、何为同步

在讨论所谓的**同步**概念之前，我们再来回顾经典的医院就诊的场景。

> 话说冤家路窄的开和赵子龙在林间相遇，话不投机三句开打。数百个来也不见胜负，但两人都已经披红挂彩，只好休战同去医院看大夫。

众所周知，医院向来是医生少患者多，经常人满为患，就算武功高强也得讲究先来后到。于是，赵子龙和铠先在挂号处取了号，一看就诊室有人就诊，外面还有长队，只好面面相觑加入到等候的队伍中。

说到这，相信你对医院这个机制并不陌生，是不是很简单？但我们要说的是，这个场景体现的就是经典的 **同步机制** ，理解这个机制你对 AQS就已经理解了一半。不信？往下看。



![image-20220417161352574](https://writting.oss-cn-beijing.aliyuncs.com/image-20220417161352574.png)

在并发编程中，说起“**同步**”，有些人可能总觉得这个词有些别扭。没错，所谓同步是`synchronize` 的翻译。但是，对于这个英文单词，我们的理解往往又不是很准确，于是听起来甚至有些别扭。因此，我们首先要能正确理解这个词的含义，才能进一步明白**同步器**是个啥玩意。

在中文语境中，我们说同步，往往指的是**两个或两个以上随时间变化的量在变化过程中保持一定的相对关系**。比如，把本地的数据同步到云端，或者是从手机同步到电脑。看到没有，中文语境中的**同步**往往指的是同一事物**在不同位置的相对变化，比如手机中的照片同步到云端，照片是不是还是那个照片？注意，它说的是同一事物。

但是，在英文中`synchronize` 的含义则完全不同于我们日常所理解的**同步**，即使我们把它翻译为**“同步”**。请看英文中对`synchronize` 的解释：

![image-20220419175923019](https://writting.oss-cn-beijing.aliyuncs.com/image-20220419175923019.png)

看到英文版释义中的`at the same time`，相信你对`synchronize` 这个词的准确含义已经豁然开朗，并了然于心。**原来，所谓同步就是搞定并发的意思，是两个事件同时发生的关系**。而在计算机科学中，同步器则是用于处理同步关系的结构和算法。

基于对同步的理解，可以发现前述的医生就诊问题也是一种同步机制的设计问题，即如何处理**多名患者**同时看**一个医生**，这里就需要考虑两个关键点：医生的**就诊状态**和**患者的队列**。下面我们通过ReentrantLock来模拟这个就诊问题：

* 如果患者通过`qualificationLock.lock()`获得了就诊资格（比如此刻没有就诊），那么他可以直接就诊；
* 如果患者未能通过`qualificationLock.lock()`获得就诊资格，则患者需要进入`qualificationLock`中的排队机制进行等待。

你看，基于AQS实现的ReentrantLock用起来很简单，其解决问题的思路也很朴素，无非是我们日常也能遇到的问题，所以我们先从朴素的问题上认识AQS。

```java
/**
 * 门诊就诊（就诊+排队）同步示例
 */
public class OutpatientDemo {
    /**
     * 当前就诊资格
     */
    private final ReentrantLock qualificationLock = new ReentrantLock();

    public void 就诊() {
        qualificationLock.lock();  // 如果门诊没有人，则获取当前就诊资格，直接就诊。否则，将进入大厅队列排队等候。
        try {
            // ... 就诊中
        } finally {
            qualificationLock.unlock(); // 就诊结束离开后，释放资格。
        }
    }
}
```



## 二、同步器的设计思路

对于普遍存在的同步问题，我们可以根据具体问题设计不同的同步机制，也就是所谓的**同步器**。比如，医生通过取号机、叫号器、显示屏和等候区来解决多人排队问题，去银行办理业务也是类似的机制。当然，这些都是我们生活中的同步机制。而在Java中，我们则可以通过AQS和`synchronize`关键字来实现同步。所以，理解同步器的设计思路是后续理解AQS的关键，而当我们从现实世界的角度看程序世界时，问题则相对更容易理解。

### （一）同步器的基本组成

无论是现实世界还是程序世界，在同步器设计的设计上总体是相似的。比如，都有**状态**和**队列**，以及围绕这两个点再增加一些其他辅助机制。所以，从抽象的角度看，一个同步器应该具备以下四要素：

* **临界状态（State）**：用于确定是否可以授予访问对象（比如线程、患者等）的访问权；
* **临界状态访问控制（Access Condition）**：用于判断访问对象是否满足访问临界状态的条件。比如，当就诊室状态为“就诊中”时，则不允许就诊人进入，反之则允许；
* **临界状态可变更（State Changes）**：当线程获得了对临界区的访问权限，它必须更改同步器的状态，以(可能)阻止其他线程进入它。换句话说，状态需要反映线程现在正在临界区内执行的事实。比如，当就诊室中有人就诊时，其状态就必须为“就诊中”，而就诊结束时则应该切换到“待就诊”；
* **状态变更可通知（Notification Strategy）**：当线程变更了同步器的状态时，应当以合适的策略通知到其他线程。比如，当前就诊人结束就诊离开诊室后，应当通过屏幕或广播等方式通知队列中等待的其他就诊人。

当然，这四要素只是在设计同步器时的基本思路和原则，具体同步器的设计并不局限于以上的四要素，可能没有其中的部分组成，也可能会有其他的组成等。

![image-20220417160430440](https://writting.oss-cn-beijing.aliyuncs.com/image-20220417160430440.png)

### （二）队列的选型

现在，我们已经理解设计同步器的要素，以及其必要的组成，比如状态和队列。对于状态，相对比较简单也好理解，比如AQS中的状态只是一个int类型的字段。而对于队列的理解，则是关键所在，如何设计有效的排队机制，关乎到数据的安全性、同步的效率以及扩展性等多方面的考量。所以，这部分我们要讲的就是如何设计队列，包含无队列和CLH队列等。

#### 1. 无队列

所谓无队列设计，即下面的左图所示，不同的线程为了争取到自己需要的资源会一拥而上，毫无秩序可言。在无队列设计中，力气大、脸皮厚的线程可能会优先获得资源，而素质高的线程则可能一直在礼让，导致始终无法获得资源，也就是我们常说的“**饥饿**”，因为它不公平。同时，无队列也会导致大量的线程阻塞，对于系统来说易引发灾难。



![](https://writting.oss-cn-beijing.aliyuncs.com/2021/06/24/16245404441166.jpg)

无队列下的线程竞争情况可以用下面这幅图来说明，每个线程通过**自旋锁**来请求资源。所谓自旋，并不是蒙上眼睛原地转圈的意思，而是它会主动地、时不时地去询问资源是否就绪，就像我们时不时跑到门口询问“**到我了吗**”。

自旋锁的优点在于实现简单，也不需要复杂的调度和队列管理。但是，它的缺点则在于：

* **锁饥饿**：在锁的竞争中，有的线程可能会始终被插队，导致饥饿；
* **性能堪忧**：多个线程同时轮询状态时，一个是消耗线程所在的CPU资源，一个是导致多个CPU的高速缓存频繁同步，影响CPU效率。

所以，**自旋锁适用于竞争不激烈、锁持有时间短的场景**，像AQS这种为各场景提供基础同步能力的同步器，自然不适合采用这种方式。

![image-20220419164936239](https://writting.oss-cn-beijing.aliyuncs.com/image-20220419164936239.png)



**既然无队列会存在无序竞争的情况，那如何解决这个问题？如果是我们来设计，如何设计这个队列，如何保证线程的公平性并兼顾性能？这就要说说CHL队列了。**

#### 2. 理解CLH队列锁

**CLH队列锁**是由Craig, Landin, and Hagersten三人共同设计的一种队列锁机制，我们先来看看CLH队列的基本模样，如下图所示。

![image-20220422202346791](https://writting.oss-cn-beijing.aliyuncs.com/image-20220422202346791.png)

从图中可以看到，在CLH队列中每个线程都是一个节点（Node），在Node中同时还有节点的前驱指针。从结构上看，CLH队列和普通队列似乎没有区别。**但是，这个队列需要解决一个核心问题：如何解决锁的竞争和释放中的性能问题。**所有节点都要自主去竞争？显然不可能。这好比我们在候诊区等待时，时不时都去问问医生”到我了吗“，如果这么做我们没累瘫，医生已经被烦死。

解决这个问题有两个思路：**一个是广播通知下一位患者，比如医院的大屏和喇叭通知；另一个则是我们可以看排在我们前面的人，比如排在我们前面的人已经进去就诊了，那下一位必然是轮到我，不然还有谁？**这两种思路，医院信息化系统通常采用的是第一种，而计算机中的CLH队列锁采用的则是第二种：**既然大家都轮询同一把锁效率低，那我们为何不能只轮询各自的前序节点状态呢**？这就是CLH队列锁的精髓所在。

现在，我们结合下面这幅图用计算机的语言来描述它的原理和精髓：**所谓CLH锁是通过移动尾部节点实现的FIFO队列，每个节点包含了线程、前序节点信息以及各自的状态。CLH队列中的各节点不会轮询同一个共享变量，而是仅轮询各自的本地变量，从而解决效率的问题。此外，FIFO属性也保障了CLH队列的公平性。**

![image-20220422202312291](https://writting.oss-cn-beijing.aliyuncs.com/image-20220422202312291.png)

当然，要实现CLH队列锁的高效和公平性，我们在构建队列时，就需要考虑如何设计队列、如何入队、如何出队和如何唤醒节点等一系列问题，而这些问题正是AQS中的一些关键问题，我们将在后文中逐渐展开叙述。



> **补充信息**：现代计算机通常是多CPU架构，各自CPU有着自己的高速缓存。当不同的线程位于不同的CPU时，线程间交换数据时就需要特别考虑性能问题。CLH队列锁针对某些架构是高效的，但换了其他架构则未必，因此我们要了解这部分知识和它的局限性。这里没有说具体的哪种架构，是为避免在本文将问题扩大化，引入太多额外知识会增加不必要的负担，有兴趣的同学可以自行检索这方面的知识。
>
> * CLH队列锁扩展阅读：https://www.cs.tau.ac.il/~shanir/nir-pubs-web/Papers/CLH.pdf
>

#### 2.3 AQS中的CLH队列锁

现在，我们知道CLH队列锁有着效率高、公平和实现简单等优点，那它有没有缺点呢？当然有。CLH队列的主要缺点在于：**一是节点的自旋影响CPU的效率；二是它的实现过于简单，不能满足AQS中复杂的场景需要**，比如AQS中节点的阻塞和唤醒等。**因此，AQS采用了CLH锁，但是对它进行了一些改造和扩展，主要是通过节点状态`waitStatus`来丰富队列的操作性**。

所以，关键的问题就来了：**如何设计重新CLH队列、如何解决入队出队等一系列问题？**

## 三、初始AQS

### （一）基本用途

AQS的全称是**AbstractQueuedSynchronizer**，即抽象队列同步器。这个名字有三个单词，其中**Abstract**表示它是一个**抽象类**，AQS源码中定义了一些具体的方法，也定义了一些抽象的方法。**换句话说，我们不能单独地直接AQS，而是需要继承它并实现部分能力**。

在JAVA的JUC中，AQS的使用非常广泛，在我们前面的系列文章都有提到，比如**Semaphore**、**ReentrantLock**、**ReentrantReadWriteLock**、**ThreadPoolExecutor**和**CountDownLatch**等，相关类对AQS的使用如下图所示。不夸张地说，在我们需要使用同步的时候，其背后几乎都有AQS的影子，只不过我们在前面都没有展开说，后文我们会选择其中两个来展开详述。

![image-20220417153748486](https://writting.oss-cn-beijing.aliyuncs.com/image-20220417153748486.png)

### （二）总体结构

在前面的系列文章中，我们已经了解设计同步器时的四大核心要素。在AQS中，也仍然遵循这几要素。因此虽然AQS看齐来很复杂，源码洋洋洒洒几千行，但是如果我们分析它的数据结构和方法会发现，其核心就是**<u>State</u>**和**<u>Queue(Node)</u>**这两个，其他都是围绕它们俩展开。所以，我们理解AQS务必要抓住其核心所在，理解其内核和外围，不要眉毛胡子一把抓，更不要慌。

![image-20220417160750212](https://writting.oss-cn-beijing.aliyuncs.com/image-20220417160750212.png)

接着上面的分析，我们来对AQS的源码和设计进行归类和总结。为了更好的结构化理解AQS，我们可以将其拆解为下面的5层。很显然，AQS中的方法众多，因此我们将相关的方法进行归类，这样在理解时会有侧重点。其中，**绿色部分**为需要重点关注的方法，**波浪线打底**的方法是需要子类实现的方法，而**实线打底**的方法则是AQS中提供的重要方法。

![image-20220425112153483](https://writting.oss-cn-beijing.aliyuncs.com/image-20220425112153483.png)

#### 1. 核心部件之状态位

作为AQS的核心之一，同步状态字段`state`的重要性不言而喻。注意，`state`字段有两点需要注意：

* `voliatile`修饰：不同线程在读写该字段时，字段的最新值对各线程可见；
* `int`类型：采用int类型是比较巧妙的设计，用于表示当前同步的状态情况。在AQS中，同步有独占和共享两种模式。其中，在独占模式下，`state`的值为`0`和`1`即可；但是，在共享模式下，`state`的值则会大于1，比如某个具有超能力加成的大夫同时可以看10个病人，那么`state`的值就为`10`。

另外，还需要注意的是，**`state`表示的是同步的状态，而不是线程的状态，线程的状态在队列的节点中，对此不要搞混淆**。

```java
/**
 * The synchronization state.
 */
private volatile int state;
```

#### 2. 核心部件之同步队列

AQS中有三个核心的字段，除了上述的`state`之外，还有两个分别是**Node**类型的`head`和`tail`. 注意，AQS并没有所谓的`queue`字段，而是用`head`和`tail`来表示队列，有头有尾，不就是队列么...那么，`head`和`tail`所构成的队列是怎样的？我们通过源码和图示来说明。

在AQS的源码中，`head`和`tail`均由`transient volatile`来修饰，这点和`state`是一致的，表示对其他线程可见。另外还需要注意的是：

* **延迟初始化（lazily initialized）**：`head`和`tail`均是在需要时才会初始化，而不是在AQS实例化即初始化。在后面我们会提到，并不是在所有情况下都需要队列，在某些情况下AQS是不需要创建的，所以它们都是延迟初始化；
* **由方法提供赋值入口**：任何线程均不可以直接修改`head`和`tail`的值，而必须通过AQS提供的方法入口来完成；

```java
/** 队列头部
 * 1. 延迟初始化；
 * 2. 经由方法提供赋值入口；
 * 3. 头部节点状态不可以为Canceled.
 */
private transient volatile Node head;

/** 队列尾部
 * 1. 延迟初始化；
 * 2. 经由方法提供赋值入口；
 */
private transient volatile Node tail;
```

AQS中由`head`和`tail`构成的队列图示如下所示。图中可以看到，`state`、`head`和`tail`构成了AQS中三个关键字段。其中，`head`和`tail`又是CHL队列的关键字段，队列中的节点通过前驱和后继的方式完成节点间关系的连接。

![image-20220512180121207](https://writting.oss-cn-beijing.aliyuncs.com/image-20220512180121207.png)

#### 3. 核心节点结构

作为AQS核心的数据结构之一，Node的组成如下图所示，包含了`waitStatus`、`thread`、`prev`、`next`和`nextWaiter`几个关键属性。其中，`waitStatus`表示当前节点线程的状态，`thread`表示当前线程，而`prev`和`next`分别代表节点的前驱和后继。

![image-20220417160809625](https://writting.oss-cn-beijing.aliyuncs.com/image-20220417160809625.png)

Node中的`waitStatus`是个重要且容易误解的属性，它有5个枚举值，为了方便理解，我们把可以这5个值以`0`为界分为三个区间来理解：

| 区间|状态|说明|
| --- |--- | --- |
| `waitStatus`=0 | 初始化状态 | 通过`new Node()`创建节点时，此时状态为0.|
| `waitStatus`>0 | 取消状态   | 线程状态无效，该线程被中断或者等待超时，需要移除该线程节点。|
| `waitStatus`<0 | 有效状态   | 包括-1、-2、-3等值。其中，-1表示该线程处于同步队列且可以被唤醒的状态，-2表示节点在条件队列中，-3用于共享模式，表示可以传播到下个节点。|

另外，关于`nextWaiter`属性，这里要先补充说明的是AQS中其实有两种类型的队列：同步队列和条件队列。我们目前主要讨论的都是同步队列，条件队列会在本文末尾讨论，而`nextWaiter`主要用于条件队列。在前面图示的同步队列中，队列是双向队列，由`prev`和`next`表示当前节点的前驱和后继，而条件队列则是单向队列，通过`nextWaiter`指向下一个节点，限于篇幅更多细节不在此处描述。

Node核心源码如下所示：

```java
static final class Node {
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

    /** 线程已经取消 */
    static final int CANCELLED =  1;
    /** 线程需要唤醒 */
    static final int SIGNAL    = -1;
    /** 线程条件等待中*/
    static final int CONDITION = -2;
    /**
     * 用于共享模式，表示可以传播到下个节点
     */
    static final int PROPAGATE = -3;

    /** 节点状态字段 */
    volatile int waitStatus;
	
  	/** 当前节点的前驱节点 */
    volatile Node prev;

	  /** 当前节点的后继节点 */
    volatile Node next;

    /** 节点中的线程 */
    volatile Thread thread;
  
    /**
    	用于条件队列，指向下一个等待节点
     */
    Node nextWaiter;

    Node() { 
    }
}
```



## 四、理解AQS的独占模式-以ReentrantLock为例

前面，我们讲述了AQS的基本组成和和核心数据结构。在这部分内容中，我们以ReentrantLock为例来分析AQS的内部机制，通过具体的示例有助于我们理解抽象的AQS。

### （一）基本用法

下面是前文所述的就诊示例代码。在这段代码中，只有当患者获得锁之后才能进入诊室就诊，否则需要排队等待，而就诊结束后则需要将锁释放，让其他等待的患者进入。示例代码只有几行，可以说是相当精简，非常容易理解。但是，其关键就在于`lock()`和`unlock()`两个方法中，**<u>这两个方法则是由ReentrantLock和AQS协作完成</u>**。所以，接下来我们在分析源码时，将会分别分析这两个部分。

```java
/**
 * 门诊就诊（就诊+排队）同步示例
 */
public class OutpatientDemo {
    /**
     * 当前就诊资格
     */
    private final static ReentrantLock qualificationLock = new ReentrantLock();

    public void 就诊() {
        qualificationLock.lock();  // 如果门诊没有人，则获取当前就诊资格，直接就诊。否则，将进入大厅队列排队等候。
        try {
            // ... 就诊中
        } finally {
            qualificationLock.unlock(); // 就诊结束离开后，释放资格。
        }
    }
}
```

### （二）加锁解析

#### 1. ReentrantLock部分源码解析

下面是精简后的ReentrantLock的`lock()`源码，我们移除了和`lock`无关的注释和其他源码，以减少对理解的干扰。在源码中，我们可以看到一些关键信息：

* ReentrantLock实现了`Lock`接口；

* ReentrantLock中的核心属性是`sync`，**而Sync类是其内部类，并继承了AQS**；

此外，我们还注意到ReentrantLock提供了**公平**和**非公平**两种模式 ，**默认是非公平模式**，所以前述示例代码使用的是非公平模式。那如何区别所谓的公平和非公平？从源码中，我们可以清晰地看到在非公平模式下，线程在请求锁时并不是立即排队，而是通过`compareAndSetState`尝试加锁，如果失败再去排队。这就像有人总爱找关系直接去找医生，但是被拒绝之后又乖乖去排队，**没有直接排队就是不公平**。

注意，在这个过程中，<u>线程尝试加锁是ReentrantLock中的源码</u>，<u>而失败后通过`acquire`排队时则将进入AQS的源码</u>。

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {        
      	abstract void lock();
    }

    static final class NonfairSync extends Sync {
        final void lock() {
            if (compareAndSetState(0, 1)) // 自己先处理
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1); // 不行再调用AQS方法
        }
    }

    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public void lock() {
        sync.lock();
    }
}
```

以上就是ReentrantLock加锁时的内部关键源码，比较简单，主要是融合**Lock接口**和**AQS同步器**。接下来，我们再顺着`acquire`方法进入AQS的源码中一探究竟。

#### 2. AQS部分源码解析

`acquire`是AQS资源抢占的重要方法入口，但它源码也很简短，只有三行。然而，通过这三行代码，我们可以看出其中的重要过程：

* 排队前不死心，通过`tryAcquire`尝试再次抢占锁。虽然成功的概率有限，**但是万一成功就没队列和排队什么事了，岂不痛快**？！
* `tryAcquire`抢占失败后，乖乖去排队；
* 排队时，先将当前线程通过`addWaiter`方法封装成一个节点，再通过`acquireQueued`方法将自己加入到队列中；
* 如果抢占失败，排队也失败，**则彻底死心，就地中断自己**。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

以上这几行便是`acquire`的核心要义，理解要义之后我们再来分析`addWaiter`等其中的细节。**这里再次提醒，通过`acquire`抢占锁时并不是总要排队的，只有当抢占失败后才会排队**。<u>**换句话说，并不是AQS中的所有场景都需要使用到队列**</u>。

**通过addWaiter源码理解入队逻辑**

`addWaiter`的主要职责是根据指定的模式将当前线程封装成节点并**入队**。这里的关键在于：

* **封装节点前要指定模式**：独占或者共享。我们示例中使用的是**独占模式（exclusive）**。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 先尝试快速入队，如果失败再通过enq走常规入队模式
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

从寥寥无几的源码中，我们可以看到入队的核心方法是`enq`，但是`addWaiter`在调用`enq`之前，**会先尝试直接加入到队尾的方式直接入队，如果失败再执行`enq`**。那为什么要这么做呢？

**<u>原因是为了提高入队效率</u>**。因为`enq`入队使用的是`for`循环的方式，所以为了避免进入循环，那自然是能直接入队最好。

比如，当赵子龙准备排队时，赵子龙看到排在队尾的是**孙尚香**，如果子龙走到队伍时队尾仍然是她，则子龙可以快速加入队伍。**但是，假如子龙走到队伍时队尾是妲己，妲己的前面才是孙尚香**。很显然，几步路的功夫队伍已经发生了变化。这个时候，如果子龙强行插队排到孙尚香的后面，可能会享受来自妲己的三连魔法攻击。**那怎么办呢?**

**当然是遵守江湖规矩，按规矩排队到队尾，拒绝插队**。而这，便是`enq`和`acquireQueued`的核心要义：**坚持按规矩排到队列的末尾，没有队列那就创建队列**。

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // 队列不存在，需要初始化
		        // 思考：这里为什么要设置一个空的节点作为头结点？稍后解释
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

#### 3. 队列变化解析

关于`enq`和`acquireQueued`入队时的核心队列变化，为了便于你的理解，我们制作了下面这幅图，图中的四个步骤反映的是队列的变化过程，我们以线程`t1`入队来分析这个过程：

1. **初始状态下，AQS队列中的`head`和`tail`指向的都是NULL;**

2. **构建首节点，这个节点是个空节点，队列的头部指向首节点；**

3. **队列的尾部指向首节点；**

4. **构建线程`t1`节点，并将t1节点的前驱指向首节点，而尾部节点则指向t1节点。**

   ![image-20220509144244096](https://writting.oss-cn-beijing.aliyuncs.com/image-20220509144244096.png)

**此处需要特别注意的是，在AQS同步队列中有一个所谓的 <u>首节点</u>，在初始化队列时会首先创建这个首节点。这个节点的存在非常有意思，因为它其实是个空节点（thread=null），但又极其容易被误解却又经常被问起**。

看到这里，我们不禁要问既然是空节点，那它存在的意义是什么？说起这个问题，我们还是要回到前述的就诊排队问题。

**理解AQA同步队列中的首节点【重要】**

我们首先来思考一个问题：**在排队就诊的队列中，真的是所有人都在排队吗**？如果是，那医生在干嘛呢？如果不是，那队伍最前面的那个人是什么状态？那究竟是还是不是？**<u>当然不是，进入首节点意味着出队</u>**。

在排队的就诊队列中，最前面的那个人当然不是在排队，<u>**他在医生那里就诊**</u>。对不对？**<u>所以，在AQS中，这个正在就诊的人就是那个空的首节点，是他锁定了资源，他的线程是正在运行中的，而不是等待中</u>**。这就是首节点，它不是排队中的节点，这点非常容易误解。

![image-20220509153500427](https://writting.oss-cn-beijing.aliyuncs.com/image-20220509153500427.png)

接下来，我们从`enq`和`acquireQueued`的源码中理解这点。在`enq`源码中，我们可以看到到当队列不存在（`t==null`）时，会执行`compareAndSetHead(new Node())`来设置空的首节点。

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {
            if (compareAndSetHead(new Node()))// 设置空的首节点
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
而在`acquireQueued`入队时，会执行`tryAcquire(arg)`来尝试抢占资源，如果成功则会执行`setHead(node)`将自己设置为首节点。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
            		// 设置当前节点抢占资源成功，设置为首节点
                setHead(node);
                p.next = null; 
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
那么，这个`setHead(node)`做了什么呢？下面的源码已经写得清清楚楚。源码只有三行，但是我们要特别注意的是，**不要认为`setHead(node)`意味着只是排到了队列的头部，它其实意味着出队、出队和出队**！<u>所以，这个方法通常是由`acquire`成功后调用</u>。

而一旦抢占资源成功后，则不需要再排队，所以在当前节点变为首节点后，会通过第二行的`node.thread = null`将其线程置为空。如此，**当该线程执行结束后，当前节点不再有其他引用，可以辅助垃圾回收**。

```java
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

#### 4. 总体流程图示

以上就是ReentrantLock中关于加锁的整体过程。在这个过程中，由ReentrantLock和AQS两部分代码共同完成。理解加锁的过程，重点在于理解其中的核心思想和步骤，比如哪些是由ReentrantLock完成的、哪些是AQS完成的、队列是如何设计的、首节点的意义是什么等等。

为了方便你理解加锁的过程，我们制作了下面这幅图，图中展示了一些过程中的一些关键步骤。

![image-20220417160728078](https://writting.oss-cn-beijing.aliyuncs.com/image-20220417160728078.png)

### （三）锁的释放解析

在上部分内容中，我们通过ReentrantLock分析了AQS的加锁过程，在这部分我们仍然结合两者再来探索AQS的解锁过程，即当我们执行`qualificationLock.unlock()`时发生了什么。

#### 1. ReentrantLock部分源码解析

ReentrantLock中的解锁入口是其`unlock`方法，在这个方法中我们可以看到ReentrantLock本身没有其他的处理逻辑，而是直接调用了AQS的`release`方法。但是，AQS的`release`会调用`tryRelease`，而`tryRelease`的实现则是由ReentrantLock实现，所以ReentrantLock源码中我们要重点关注的只有`tryRelease`这个方法。

`tryRelease`在实现上并不复杂，关键点在于当状态为`0`时则视为释放成功，而且非当前抢占线程不允许释放。

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;
  
	  abstract static class Sync extends AbstractQueuedSynchronizer {
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) { // 释放成功
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
    }

    public void unlock() {
        sync.release(1);
    }
}
```

#### 2. AQS部分源码解析

AQS源码部分的重点则在于`release`方法，它主要调用子类的`tryRelease`，当子类判定释放成功后，则唤醒后继节点线程。

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) { // 调用子类实现
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); // 唤醒后继节点
        return true;
    }
    return false;
}
```
`unparkSuccessor`方法主要用于唤醒后继节点。**需要注意的是，在唤醒过程中可能有的节点已经取消或者为null，那么这个时候会依次向后寻找有效的节点来唤醒**。

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
        
    Node s = node.next; // 找到待唤醒的后继节点
    if (s == null || s.waitStatus > 0) { 
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev) // 后继节点可能为null或状态错误，则循环向后查找
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null) // 找到有效的后继节点，唤醒它
        LockSupport.unpark(s.thread);
}
```

#### 3. 总体流程图示

下面是ReentrantLock释放锁时的流程图，相对于加锁而言，锁的释放要简单些。流程中的重点在于要区分哪些是ReentrantLock完成的，哪些是AQS完成的，**以及如何唤醒后继节点**。

![image-20220418120751848](https://writting.oss-cn-beijing.aliyuncs.com/image-20220418120751848.png)

## 五、理解AQS的共享模式-以Semaphore为例

现在，我们已经知道，AQS中的核心属性之一`state`是个`int`类型的变量，并且AQS的同步状态支持**独占模式**和共享模式，而在ReentrantLock中我们已经理解AQS在独占模式下的工作原理，**所以在这部分我们将借助Semaphore来理解AQS中的共享模式**。（如果你对Semaphore不甚了解，可以查阅王者并发课系列的相关专题文章）

### （一）基本用法

通常，我们在医院就诊时往往是一个诊室同时仅问诊一个患者，所以我们可以通过ReentrantLock来模拟这一场景。**但是，假如门诊里有两名医生，可以同时问诊两名患者**。那么，这种场景下我们就可以使用Semaphore来模拟，而Semaphore的背后正是AQS的共享模式。相关示例代码如下所示，诊室同时允许两名患者进入。

```java
/**
 * 门诊就诊（就诊+排队）同步示例
 */
public class OutpatientDemo {
    /**
     * 当前就诊资格，允许2人同时就诊
     */
    private final static Semaphore permits = new Semaphore(2);

    public void 就诊() {
        permits.acquire();  // 如果当前有可用就诊资格，则获取当前就诊资格，直接就诊。否则，将进入大厅队列排队等候。
        try {
            // ... 就诊中
        } finally {
            permits.release(); // 就诊结束离开后，释放资格。
        }
    }
}
```

那么，当代码执行`permits.acquire()`时发生了什么？我们接着往下看。

### （二）资源抢占解析

Semaphore在执行`permits.acquire()`时分两部分完成，一部分在Semaphore中完成，一部分则由AQS完成。

#### 1. Semaphore部分源码解析

以下是Semaphore中关于`acquire()`的核心源码，为了减少其他代码对你的影响，我们已经删除了不必要的注释和其他代码，仅保留和`acquire()`相关的代码。

在代码中，我们可以看到Semaphore支持公平和非公平两种模式，在ReentrantLock的示例中我们使用的非公平模式，而在此我们将示例Semaphore的公平模式，但其实两者差别并不大。关于下面的源码部分，有几个关键点需要我们注意：

* 默认情况下Semaphore使用的是非公平模式；
* `permits.acquire()`运行时，调用的是AQS中的`sync.acquireSharedInterruptibly(1)`方法，而这个方法又会反过来调用Semaphore中的`tryAcquireShared`方法；
* 源码中`acquire()`、`acquire(1)`、`tryAcquire()`和`acquireUninterruptibly()`等都只不过是中断、超时等不同的变种，其核心思路不变，不要有“乱花渐欲迷人眼”的错觉。

```java
public class Semaphore implements java.io.Serializable {
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) {
            setState(permits);
        }
    }

    static final class FairSync extends Sync {
        FairSync(int permits) {
            super(permits);
        }
				// 公平模式下获取共享资格
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

	  // 构建信号量，并初始化资格总数
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
		// 获取资格，获取失败后等待
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
		// 释放资格
    public void release() {
        sync.releaseShared(1);
    }
}
```

#### 2. AQS部分源码解析

AQS在处理共享资源申请时，也有很多不同的变种方法，比如中断、超时等，但其核心思路一致，所以这里使用`acquireSharedInterruptibly`来讲解，这个方法也是Semaphore调用的方法。

`acquireSharedInterruptibly`方法的源码如下所示，主要表达了两层含义：
* 如果当前线程中断，则抛出异常；
* 如果Semaphore中的`tryAcquireShared`返回的结果小于0，则意味着可以继续，将进入AQS队列处理逻辑。

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
    		// 存在可用资源，进入共享获取逻辑
        doAcquireSharedInterruptibly(arg);
}
```
AQS共享模式下的队列处理逻辑和独占模式下的处理逻辑总体相似，其核心差异在于`addWaiter(Node.SHARED)`和`setHeadAndPropagate(node, r)`两行代码，前者标记了当前节点为共享模式，而后者则在将自己设置为头部后，同时唤醒后继节点。那么，唤醒后继节点是什么意思？

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 新节点设置为共享模式
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
            		// 调用子类的方法
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                		// 设置头部和传播状态
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

这是因为，在AQS的独占模式中，资源`state`同时仅允许一个线程抢占，所以除了抢占成功的节点，其他节点均处理等待状态。**但是，在AQS的共享模式中，虽然当前线程抢占了资源，但它抢占的仅是部分资源，还可能有剩余资源可被其他线程抢占，所以它要通过`setHeadAndPropagate(node, r)`唤醒其他节点线程**。

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 如果首节点状态是SIGNAL，则需要unpark唤醒下个节点线程
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // 循环检查，多次确认
                // 唤醒首节点下个节点线程
                unparkSuccessor(h);
            }
            // ws==0时，将会尝试将ws设置为Node.PROPAGATE，这样setHeadAndPropagate读到ws<0时，就会唤醒后继节点
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // 循环检查，防止头部发生变化
            break;
    }
}
```

#### 3. 总体流程图示

![image-20220512180047272](https://writting.oss-cn-beijing.aliyuncs.com/image-20220512180047272.png)

### （三）资源释放解析

#### 1. Semaphore部分源码解析

Semaphore通过`release()`执行资源释放，如下源码所示。`release()`直接调用AQS中的`releaseShared()`，当然这并不是说释放资源都是AQS的事而与Semaphore无关，因为`releaseShared()`中会调用Semaphore中的`tryReleaseShared()`。

```java
public class Semaphore implements java.io.Serializable {
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) {
            setState(permits);
        }
        
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // 如果传入的releases数值为负，则抛出异常
                    throw new Error("Maximum permit count exceeded");
                // 更新最新值
                if (compareAndSetState(current, next))
                    return true;
            }
        }
    }

    public void release() {
        sync.releaseShared(1);
    }
}
```

#### 2. AQS部分源码解析

下面是`releaseShared()`方法的源码，可以看到它只做了两件事：一是调用模板方法`tryReleaseShared()`，二是调用`doReleaseShared()`，前者由子类Semaphore提供，后者由AQS提供。

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

`doReleaseShared()`方法是AQS共享模式下的关键方法，其实它在前面的`setHeadAndPropagate()`方法中也有引用，其关键源码如下。

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 如果首节点状态是SIGNAL，则需要unpark唤醒下个节点线程
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // 循环检查，多次确认
                // 唤醒首节点下个节点线程
                unparkSuccessor(h);
            }
            // ws==0时，将会尝试将ws设置为Node.PROPAGATE，这样setHeadAndPropagate读到ws<0时，就会唤醒后继节点
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // 循环检查，防止头部发生变化
            break;
    }
}
```


#### 3. 总体流程图示

![image-20220510180435980](https://writting.oss-cn-beijing.aliyuncs.com/image-20220510180435980.png)

## 六、理解AQS中的条件同列

前述内容所讲的都是AQS中的同步队列，接下来我们再来探讨AQS中的条件队列。为了便于理解，我们仍然以医院就诊为例。

在医院候诊排队时，正常情况下按照队伍排队即可。然而，凡事都有例外，**比如防疫规定没有做核酸的不能到大厅排队，要现在外面做核酸并在结果出来后才能排队**。此时，就会出现两个队列：**一是按先来后到的正常候诊队列，二是核酸结果等待队列**。

![image-20220519164637626](https://writting.oss-cn-beijing.aliyuncs.com/image-20220519164637626.png)

这样的医院排队机制相信你可以理解。其实，这种机制不仅在现实中存在，在软件中也存在。作为强大的同步器，AQS所提供的条件队列正是为了解决此类问题。我们将可以将正常候诊的队列称为**同步队列**，而需要等待核酸结果出来后才能进入排队的称为**条件队列**。

所以，基于目前的理解，我们可以进一步完善AQS的整体结构：**同步状态+同步队列+条件队列**，如下图所示。

AQS的条件队列是**单向队列**，由`ConditionObject+Node`组成，ConditionObject中定义了队列的**头部（firstWaiter）**和**尾部（lastWaiter）**，并且头部和尾部都是Node类型。和AQS中的同步队列类似，条件队列也有着广泛的使用。

![image-20220512180900730](https://writting.oss-cn-beijing.aliyuncs.com/image-20220512180900730.png)

比如，ReentrantLock中的`newCondition()`方法所调用的就是AQS中的方法，而ArrayBlockingQueue则使用了ReentrantLock这一方法，相关使用如下所示。

* ReentrantLock中的方法

```java
public Condition newCondition() {
    return sync.newCondition(); // 创建新的等待条件
}
```

* ReentrantLock所调用的AQS方法：
```JAVA
final ConditionObject newCondition() {
    return new ConditionObject(); // 构建等待队列
}
```
* ArrayBlockingQueue中所使用的ReentrantLock中的方法，比如`lock.newCondition()`:
```java
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition(); // 等待队列的具体场景应用
    notFull =  lock.newCondition();
}
```

* ArrayBlockingQueue中通过`notFull.await()`将当前线程放入条件队列，等待唤醒。
```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await(); // 条件队列中的等待
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```

需要注意的是，**同一线程不可以同时处于同步队列和条件队列中**。**当线程进入条件队列时，将自动从原来的同步队列中出队**。限于篇幅，关于条件队列的更多源码和流程在此不做更多解释，读者可以自行探索。

## 小结

正文到此结束，恭喜你又上了一颗星✨

在本文中，我们首先厘清了同步的概念，无论是现实生活还是软件设计，同步都是广泛的存在，而从生活中理解软件中的设计相对较为容易。对于同步问题的解决，队列是常被采用的方案。但是在软件设计中，队列的设计需要考虑到**公平性、性能和扩展性**等多个维度，所以虽然队列是AQS的核心组件之一，但是对CLH队列进行了适当的改造，以更好地适配AQS的设计理念和需求。因此，理解AQS的核心在于理解变种的CLH队列，包括它的设计理念、数据结构组成，以及出队和入队等完整过程，所以我们在开篇引入并介绍了CLH队列。

在本文的第四和第五部分，我们以ReentrantLock和Semaphore为例，介绍了AQS独占模式和共享模式下的入队和队列的变化形态，重点还是在于帮助理解CLH队列。而在第六部分，我们介绍了**似乎鲜为人知但同样重要**的条件队列。

本文整体篇幅较长，内容较多。然而，在理解AQS时，我们不要深陷冗长的文章和源码中。首先要清楚的并非AQS是什么和它的工作原理，而是要**先搞清楚AQS所解决的是什么问题，**针对问题AQS提出了怎样的方案。如此，才能抓住AQS的核心脉络，理解它的本质。

另外，作为成熟的同步器，AQS提供了完善的各种同步机制，JDK中也提供了多样的同步实现，比如ReentrantLock、Semaphore和CountDownLatch等。**因此，在编码中需要使用同步机制时，应首先考虑现有的稳定的同步方案，其次再考虑自由地自主实现**。

## 夫子的试炼

* 基于AQS，设计自己的同步器：实现一个队列，三个窗口同时核酸采样。

## 延伸阅读与参考资料

* http://tutorials.jenkov.com/java-concurrency/anatomy-of-a-synchronizer.html
* https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html
* https://blog.51cto.com/u_12302616/3230929
* https://www.cnblogs.com/xijiu/p/14396061.html
* https://www.javarticles.com/2012/10/abstractqueuedsynchronizer-aqs.html
* https://www.infoq.cn/article/bvpvyvxjkm8zstspti0l
* https://www.cs.tau.ac.il/~shanir/nir-pubs-web/Papers/CLH.pdf
* 掘金专栏：https://juejin.cn/column/6963590682602635294
* Github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 常见面试题

* 说说自己对 AQS 的理解？
* 多个线程通过锁请求共享资源，获取不到锁的线程怎么办？
* 独占模式和共享模式有哪些区别？
* 同步队列中，线程入、出同步队列的时机和过程？
* 为什么 AQS 有了同步队列之后，还需要条件队列？
* 如果一个线程需要等待一组线程全部执行完之后再继续执行，有什么好的办法么？是如何实现的？

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。
