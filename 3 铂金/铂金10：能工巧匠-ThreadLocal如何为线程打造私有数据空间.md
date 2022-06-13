欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第23篇**，铂金中的**第10篇**。

说起**ThreadLocal**，相信你对它的名字一定不陌生。在并发编程中，它有着较高的出场率，并且也是面试中的高频面试题之一，所以**其重要性不言而喻**。当然，它也可能曾经让你在夜里辗转反侧，或让你在面试时闪烁其词。**因为，ThreadLocal虽然使用简单，但要理解它的原理又似乎并不容易**。

然而，正所谓**明知山有虎，偏向虎山行**。在本文中，我将和你一起学习ThreadLocal的用法及其原理，啃下这块硬骨头。

关于ThreadLocal的用法和原理，网上也有着非常多的资料可以查阅。遗憾的是，这其中的大部分资料在讲解时都不够透彻。有的是蜻蜓点水，没有把必要的细节讲清楚，有的则比较片面，只讲了其中的某个点。

所以，当王者并发课系列写到这篇文章的时候，**如何才能简明扼要地把ThreadLocal介绍清楚，让读者能在一篇文章中透彻地理解它，但同时又要避免万字长文读不下去，是我最近一直在思考的问题**。为此，在综合现有资料的基础上，我精心设计了一些配图，尽可能地让文章图文并茂，以帮助你相对轻松地理解ThreadLocal中的精要。然而，每个读者的背景不同，理解也就不同。所以，对于你认为的并没有讲清楚的地方，希望你在评论区留言反馈，我会尽量调整完善，争取让你“**一文读懂**”。

## 一、ThreadLocal使用场景初体验

**夫子的疑惑：在什么场景下需要使用ThreadLocal**？

在王者峡谷中，每个英雄都有着自己的领地和庄园。在庄园里，按照功能职责的不同又划分为不同的区域，比如有**圈养野怪的区域**，还有**存放金币**以及**武器**等不同区域。**当然，这些区域都是英雄私有的，不能混淆错乱**。

所以，铠在打野和获得金币时，可以把他打的野怪放进自己庄园里，那是他的**私有空间**。同样，兰陵王和其他英雄也是如此。这个设计如下图所示：

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/07/10/16259060092066.jpg)

现在，我们就来编写一段代码模拟上述的场景：

*  **铠在打野和获得金币时，放进他的私有空间里**；
*  **兰陵王在打野和获得金币时，放进他的私有空间里**；
*  **他们的空间都位于王者峡谷中**。

以下是我们编写的一段代码。在代码中，我们定义了一个`wildMonsterLocal`变量，用于存放英雄们打野时获得的野怪；而`coinLocal`则用于存放英雄们所获得的金币。于是，铠将他所打的**棕熊**放进了圈养区，并将获得的**500金币**放进了**金币存放区**；而兰陵王则将他所打的野狼放进了圈养区，并将获得的**100金币**放进了**金币存放区**。

过了一阵子之后，他们分别取走他们存放的野怪和金币。

主要示例如下所示。在阅读下面示例代码时，要着重注意对ThreadLocal的`get`和`set`方法的调用。

```java
//私人野怪圈养区
private static final ThreadLocal < WildMonster > wildMonsterLocal = new ThreadLocal < > ();
//私人金币存放区
private static final ThreadLocal < Coin > coinLocal = new ThreadLocal < > ();

public static void main(String[] args) {
  Thread 铠 = newThread("铠", () -> {
    try {
      say("今天打到了一只棕熊，先把它放进圈养区，改天再享用！");
      wildMonsterLocal.set(new Bear("棕熊"));
      say("路上杀了一些兵线，获得了一些金币，也先存起来！");
      coinLocal.set(new Coin(500));

      Thread.sleep(2000);
      note("\n过了一阵子...\n");
      say("从圈养区拿到了一只：", wildMonsterLocal.get().getName());
      say("金币存放区现有金额：", coinLocal.get().getAmount());
    } catch (InterruptedException e) {}
  });

  Thread 兰陵王 = newThread("兰陵王", () -> {
    try {
      Thread.sleep(1000);
      say("今天打到了一只野狼，先把它放进圈养区，改天再享用！");
      wildMonsterLocal.set(new Wolf("野狼"));
      say("路上杀了一些兵线，获得了一些金币，也先存起来！");
      coinLocal.set(new Coin(100));

      Thread.sleep(2000);
      say("从圈养区拿到了一只：", wildMonsterLocal.get().getName());
      say("金币存放区现有金额：", coinLocal.get().getAmount());
    } catch (InterruptedException e) {}
  });
  铠.start();
  兰陵王.start();
}

```

示例代码中用到的类如下所示：

```java
@Data
private static class WildMonster {
  protected String name;
}

private static class Wolf extends WildMonster {
  public Wolf(String name) {
    this.name = name;
  }
}

private static class Bear extends WildMonster {
  public Bear(String name) {
    this.name = name;
  }
}

@Data
private static class Coin {
  private int amount;

  public Coin(int amount) {
    this.amount = amount;
  }
}
```

示例代码运行结果如下：

```shell
铠:今天打到了一只棕熊，先把它放进圈养区，改天再享用！
铠:路上杀了一些兵线，获得了一些金币，也先存起来！
兰陵王:今天打到了一只野狼，先把它放进圈养区，改天再享用！
兰陵王:路上杀了一些兵线，获得了一些金币，也先存起来！

过了一阵子...

铠:从圈养区拿到了一只：棕熊
铠:金币存放区现有金额：500
兰陵王:从圈养区拿到了一只：野狼
兰陵王:金币存放区现有金额：100

Process finished with exit code 0
```

从运行的结果中，可以清楚地看到，在过了一阵子之后，**铠和兰陵王分别取到了他们之前存放的野怪和金币，并且丝毫不差**。

**以上，就是ThreadLocal应用的典型**。在多线程并发场景中，如果你需要为**每个线程**设置可以**跨越类和方法**层面的私有变量，那么你就需要考虑使用ThreadLocal了。注意，这里有两个要点，**一是变量为某个线程独享，二是变量可以在不同方法甚至不同的类中共享**。

ThreadLocal在软件设计中的应用场景非常多。举个简单的例子，在一次请求中，如果你需要设置一个**traceId**来跟踪请求的完整调用链路，那么你就需要一个能跨越类和方法的变量，这个变量可以让线程在不同的类中自由获取，且不会出错，其过程如下图所示：

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/07/11/16259392612164.jpg)


## 二、ThreadLocal原理解析

对于ThreadLocal，一般来说被提及最多的可能就是那个经典的面试问题：**谈谈你对ThreadLocal内存泄露的理解**。这个问题看起来很简单，但要回答到点子上的话，就必须对其源码有足够理解。当然，背诵面试题的答案扯一通“**软引用**”、“**内存回收**”巴拉巴拉也是可以的，毕竟大部分的面试官也是半吊子。

接下来，我们会结合上文的场景，以及它的示例代码来讲解ThreadLocal的原理，让你找到关于这个问题的真正答案。

### 1. 源码分析

如果对ThreadLocal理解有困难的话，很大的可能是：**你没有理清不同概念之间的关系**。所以，**理解ThreadLocal源码的第一步是找出它的相关概念，并理清它们之间的关系，即Thread、ThreadLocalMap和ThreadLocal**。正是这三个关键概念，唱出了一台好戏。当然，如果细分的话，你也可以把**Entry**单独拎出来。

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/07/11/16259347326996.jpg)


**关键概念1：Thread类**

为什么Thread在关键概念中排名第一，因为ThreadLocal就是为它而生的。那Thread和ThreadLocal是什么关系呢？我们这就来看看Thread的源码：

```java
class Thread implements Runnable {
    ...
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ...
}
```
没有什么比源码展现更清晰的了。你可以非常直观地看到，Thread中有一个变量：`threadLocals`. **通俗地说，这个变量就是用来存放当前线程的一些私有数据的，并且可以存放多个私有数据**，毕竟线程是可以携带多个私有数据的，比如它可以携带`traceId`，也自然可以携带`userId`等数据。理解了这个变量的用途之后，再看看它的类型，也就是`ThreadLocal.ThreadLocalMap`.你看，Thread就这样和ThreadLocal扯上了关系，所以接下来我们来看另外一个关键概念。

**关键概念2：ThreadLocalMap类**

从Thread的源码中你已经看到，Thread是用ThreadLocalMap来存放线程私有数据的。这里，我们先暂且撇开ThreadLocal，来直接看ThreadLocalMap的源码：

```java
static class ThreadLocalMap {
        
        ...

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;
        ...

}
```

ThreadLocalMap中最关键的属性就是`Entry[] table`，正是它实现了线程私有多数据的存储。而Entry则是继承了WeakReference，并且Entry的Key类型是ThreadLocal. 看到这里，先不要想着ThreadLocalMap的其他源码，你现在应当理解的是，**`table`是线程私有数据存储的地方，而ThreadLocalMap的其他源码不过都是为了`table`数据的存与取而存在的**。这是你对ThreadLocalMap理解的关键，不要把自己迷失在错综复杂的其他源码中。

**关键概念3：ThreadLocal类**

现在，目光终于到了ThreadLocal这个类上。Thread中使用到了ThreadLocalMap，而接下来你会发现ThreadLocal不过是封装了一些对ThreadLocalMap的操作。你看，ThreadLocal中的`get()`、`set()`、`remove()`等方法都是在操作ThreadLocalMap. 在各种操作之前，都会通过`getMap()`方法拿到当前线程的ThreadLocalMap.

```java
public class ThreadLocal<T> {

    ...

    // 获取当前线程的数据
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    // 初始化数据
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }

    // 设置当前线程的数据
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }

    // 获取线程私有数据存储的关键，虽然操作在ThreadLocal中，但是实际操作的是Thread中的threadLocals变量
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    
    // 初始化线程的t.threadLocals变量，设置为空值
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    ...
}
```

如果，此时你对相关概念及其源码的理解仍然感到困惑，**那就对了**。下面这幅图，将结合相关概念和示例代码，来还原这其中的相关概念和它们之间的关系，这幅图值得你反复细品。


![](https://writting.oss-cn-beijing.aliyuncs.com/2021/07/10/16259060371310.jpg)

在上面这幅图中，你需要如下这些细节：

* 有两个线程：**铠**和**兰陵王**；
* 有两个ThreadLocal对象，它们分别用于存放线程的私有数据，即英雄们的野怪和金币；
* **线程铠**和**线程兰陵王**都有一个ThreadLocal.ThreadLocalMap的变量，用来存放不同的ThreadLocal，即`wildMonsterLocal`和`coinLocal`这两个变量都会放进ThreadLocalMap的`table`里，也就是entry数组中；
* 当线程向ThreadLocalMap中放入数据时，它的key会指向ThreadLocal对象，而value则是ThreadLocal中的值。比如，当**铠**将**棕熊**放入`wildMonsterLocal`中时，对应Entry的key是`wildMonsterLocal`，而value则是`new Bear()`，即**棕熊**。**当兰陵王放入野怪时，同理；当铠放入金币时，也是同理**；
* 当Entry的key指向ThreadLocal对象时，比如指向`wildMonsterLocal`或`coinLocal`时，注意，**是虚引用**，**是虚引用**，**是虚引用**，**是虚引用**！重要的事情，**说四遍**。看图中的红线虚线，或ThreadLocalMap源码中的`WeakReference`.

如果你已经看明白上面这幅图，那么下面这幅图中的关系也就应该一目了既然。否则，如果你似乎看不明白它，请回到上面继续品上面那幅图，直到你对下图一目了然。

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/07/10/16259136561910.jpg)


### 2. 使用指南

接下来，将为你简单介绍ThreadLocal的一些常见高频用法。

**（1）创建ThreadLocal**

像创建其他对象一样创建即可，没有什么特别之处。

```java
ThreadLocal < WildMonster > wildMonsterLocal = new ThreadLocal < > ();
```
在对象创建完成之后，每个线程便可以向其中读写数据。当然，每个线程都只能看到它们自己的数据。


**（2）设置ThreadLocal的值**
```java
wildMonsterLocal.set(new Bear("棕熊"));
```

**（3）取出ThreadLocal的值**
```java
wildMonsterLocal.get();
```

在读取数据时需要注意的是，如果此时还没有数据设置进来，那么将会调用`setInitialValue`方法来设置初始值并返回给调用方。

**（4）移除ThreadLocal的值**
```java
wildMonsterLocal.remove();
```

**（5）初始化ThreadLocal的值**
```java
private ThreadLocal wildMonsterLocal = new ThreadLocal<WildMonster>() {
    @Override 
    protected WildMonster initialValue() {
        return new WildMonster();
    }
};   
```
在对ThreadLocal进行`get`操作时，如果当前尚未进行过数据设置，那么会执行初始化动作，如果你此时希望设置初始值，可以重写它的`initialValue`方法。

### 3. 如何理解ThreadLocal的内存泄露问题

首先，你要理解**弱引用**这个概念。在Java中，引用分为强引用、弱引用、软引用、虚幻引用等不同的引用类型，**而不同的引用类型对应的则是不同的垃圾回收策略**。如果你对此不熟的话，建议可以去检索相关资料，也可以看[这篇](https://www.cnblogs.com/fengbs/p/7019687.html)。

对于弱引用，在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，**不管当前内存空间足够与否，都会回收它的内存**。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。但是，即便是偶尔发生，也足够造成问题。

当你理解了弱引用和对应的垃圾回收策略之后，此刻，请回到上面的那幅图：
![](https://writting.oss-cn-beijing.aliyuncs.com/2021/07/11/16259373364480.jpg)

在这幅图里，Entry的key指向ThreadLocal对象时，用的正是弱引用，图中已经红色箭头标注。这里的红色虚线会造成上面问题呢？你想想看，如果此时ThreadLocal对象被回收时，那么Entry中的key就编程了null. **可是，虽然key（wildMonsterLocal）变成了null，value的值（new Bear("棕熊")）还是强引用，它还会继续存在，但实际已经没有用了，所以会造成这个Entry就废了，但是因为value的存在却不能被回收**。于是，内存泄露就这样产生了。

**那既然如此，为什么要使用弱引用？**

相信你一定有这个疑问，如果没有，这篇文章你可能需要再读一遍。明知这里会产生内存泄露的风险，却仍然使用弱引用的原因在于：**当ThreadLocal对象没有强引用时，它们需要被清理，否则它们长期存在于ThreadLocalMap中，也是一种内存泄露**。你看，问题就是这样的一环扣着一环。

**最佳实践：如何避免内存泄露**

那么，既然事已如此，如何避免内存泄露呢？这里给出一个可行的最佳实践：**在调用完成后，手动执行remove()方法**。

```java
private static final ThreadLocal<WildMonster> wildMonsterLocal = new ThreadLocal<>();

try{
    wildMonsterLocal.get();
    ...
}finally{
    wildMonsterLocal.remove();
}

```

**除此之外，ThreadLocal也给出一个方案**：在调用`set`方法设置时，会调用`replaceStaleEntry`方法来检查key为null的Entry。如果发现有key为null的Entry，那么会将它的value也设置为null，这样Entry便可以被回收。**当然，如果你没有再调用`set`方法，那么这个方案就是无效的**。

```java
private void set(ThreadLocal < ? > key, Object value) {

     ...

     for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
         ThreadLocal < ? > k = e.get();

         if (k == key) {
             e.value = value;
             return;
         }

         if (k == null) {
             replaceStaleEntry(key, value, i); //看这里
             return;
         }
     }

     ...
 }
```

## 小结

以上就是关于ThreadLocal的全部内容。在学习ThreadLocal时，首先要理解的是它的应用场景，即它所要解决的问题。其次，对它的源码要有一定的了解。在了解源码时，要注意从Thread、ThreadLocal和ThreadLocalMap三个概念出发，理解他们之间的关系。如此，你才能完全理解常见的内存泄露问题是怎么一回事。

正文到此结束，恭喜你又上了一颗星✨

## 夫子的试炼

* 尝试向你的朋友解释ThreadLocal内存泄露是如何发生的。

## 延伸阅读与参考资料

* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。

