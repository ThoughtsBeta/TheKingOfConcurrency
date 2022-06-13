欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第25篇**，砖石中的**第2篇**。

在上一篇文章中，我们学习了线程池ThreadPoolExecutor，它通过对任务队列和线程的有效管理实现了对并发任务的处理。然而，ThreadPoolExecutor有两个明显的缺点：**一是无法对大任务进行拆分，对于某个任务只能由单线程执行；二是工作线程从队列中获取任务时存在竞争情况**。这两个缺点都会影响任务的执行效率，要知道高并发场景中的每一毫秒都弥足珍贵。

针对这两个问题，本文即将介绍的**ForkJoinPool**给出了可选的答案。在本文中，我们将首先从分治算法开始介绍，接着体验ForkJoinPool中自定义任务的实现，最后再深入到Java中去理解ForkJoinPool的原理和用法。

**本文大约2万字，篇幅较长，在阅读时建议先看目录再看内容或先收藏**。

## 一、分治算法与Fork/Join模式

在并发计算中，Fork/Join模式往往用于对大任务的并行计算，它通过递归的方式对任务不断地拆解，再将结果进行合并。如果从其思想上看，Fork/Join并不复杂，其本质是**分治算法（Divide-and-Conquer）** 的应用。

分治算法的**基本思想是将一个规模为N的问题分解为K个规模较小的子问题，这些子问题相互独立且与原问题性质相同。求出子问题的解，就可得到原问题的解。**分治算法的步骤如下：

* **（1）分解**：将要解决的问题划分成若干规模较小的同类问题；
* **（2）求解**：当子问题划分得足够小时，用较简单的方法解决；
* **（3）合并**：按原问题的要求，将子问题的解逐层合并构成原问题的解。

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/07/31/16277221085724.jpg)

Fork/Join对任务的拆分和对结果合并过程也是如此，可以用下面伪代码来表示：

```java
solve(problem):
    if problem is small enough:
        // 如果任务足够小，执行任务
        solve problem directly (sequential algorithm)
    else:
        // 拆分任务
        for part in subdivide(problem)
            fork subtask to solve(part)
        // 合并结果
        join all subtasks spawned in previous loop
        return combined results
```

所以，理解Fork/Join模型和ForkJoinPool线程池，首先要理解其背后的算法的目的和思想，因为后文所要详述的ForkJoinPool不过只是这种算法的一种的实现和应用。

## 二、Fork/Join应用场景与体验

按照王者并发课所提倡的**思想->实现->源码**的思路，在了解了Fork/Join思想之后，我们先通过一个场景手工实现一个RecursiveTask，这样可以更好地体验Fork/Join的用法。

>场景：给定两个自然数，计算两个两个数之间的总和。比如1~n之间的和：1+2+3+…+n

为了解决这个问题，我们创建了TheKingRecursiveSumTask这个核心类，它继承于RecursiveTask. RecursiveTask是ForkJoinPool中的一种任务类型，你暂且不必深入了解它，后文会有详细描述。

TheKingRecursiveSumTask中定义了任务计算的起止范围（`sumBegin`和`sumEnd`）和拆分阈值（`threshold`），以及核心计算逻辑`compute()`.

```java
public class TheKingRecursiveSumTask extends RecursiveTask<Long> {
    private static final AtomicInteger taskCount = new AtomicInteger();
    private final int sumBegin;
    private final int sumEnd;
    /**
     * 任务拆分阈值，当任务尺寸大于该值时，进行拆分
     */
    private final int threshold;

    public TheKingRecursiveSumTask(int sumBegin, int sumEnd, int threshold) {
        this.sumBegin = sumBegin;
        this.sumEnd = sumEnd;
        this.threshold = threshold;
    }

    @Override
    protected Long compute() {
        if ((sumEnd - sumBegin) > threshold) {
            // 两个数之间的差值大于阈值，拆分任务
            TheKingRecursiveSumTask subTask1 = new TheKingRecursiveSumTask(sumBegin, (sumBegin + sumEnd) / 2, threshold);
            TheKingRecursiveSumTask subTask2 = new TheKingRecursiveSumTask((sumBegin + sumEnd) / 2, sumEnd, threshold);
            subTask1.fork();
            subTask2.fork();
            taskCount.incrementAndGet();
            return subTask1.join() + subTask2.join();
        }
        // 直接执行结果
        long result = 0L;
        for (int i = sumBegin; i < sumEnd; i++) {
            result += i;
        }
        return result;
    }

    public static AtomicInteger getTaskCount() {
        return taskCount;
    }
}

```
在下面的代码中，我们设置的计算区间值0~10000000，**当计算的个数超过100时，将对任务进行拆分**，最大并发数设置为**16**. 

```java
 public static void main(String[] args) {
     int sumBegin = 0, sumEnd = 10000000;
     computeByForkJoin(sumBegin, sumEnd);
     computeBySingleThread(sumBegin, sumEnd);
 }

 private static void computeByForkJoin(int sumBegin, int sumEnd) {
     ForkJoinPool forkJoinPool = new ForkJoinPool(16);
     long forkJoinStartTime = System.nanoTime();
     TheKingRecursiveSumTask theKingRecursiveSumTask = new TheKingRecursiveSumTask(sumBegin, sumEnd, 100);
     long forkJoinResult = forkJoinPool.invoke(theKingRecursiveSumTask);
     System.out.println("======");
     System.out.println("ForkJoin任务拆分：" + TheKingRecursiveSumTask.getTaskCount());
     System.out.println("ForkJoin计算结果：" + forkJoinResult);
     System.out.println("ForkJoin计算耗时：" + (System.nanoTime() - forkJoinStartTime) / 1000000);
 }

 private static void computeBySingleThread(int sumBegin, int sumEnd) {
     long computeResult = 0 L;
     long startTime = System.nanoTime();
     for (int i = sumBegin; i < sumEnd; i++) {
         computeResult += i;
     }
     System.out.println("======");
     System.out.println("单线程计算结果：" + computeResult);
     System.out.println("单线程计算耗时：" + (System.nanoTime() - startTime) / 1000000);
 }
```

运行结果如下：

```shell
======
ForkJoin任务拆分：131071
ForkJoin计算结果：49999995000000
ForkJoin计算耗时：207
======
单线程计算结果：49999995000000
单线程计算耗时：40

Process finished with exit code 0
```

从计算结果中可以看到，ForkJoinPool总共进行了131071次的任务拆分，最终的计算结果是49999995000000，耗时207毫秒。

不过，细心的你可能已经发现了，**ForkJoin的并行计算的耗时竟然比单程程还慢？并且足足慢了近5倍**！先别慌，关于ForkJoin的性能问题，我们会在后文有讲解。

## 三、ForkJoinPool设计与源码分析

在Java中，ForkJoinPool是Fork/Join模型的实现，于Java7引入并在Java8中广泛应用。ForkJoinPool允许其他线程向它提交任务，并根据设定将这些任务拆分为粒度更细的子任务，这些子任务将由ForkJoinPool内部的工作线程来并行执行，并且工作线程之间可以窃取彼此之间的任务。 

在接口实现和继承关系上，ForkJoinPool和ThreadPoolExecutor类似，都实现了Executor和ExecutorService接口，并继承了AbstractExecutorService抽类。而在任务类型上，ForkJoinPool主要有两种任务类型：**RecursiveAction**和**RecursiveTask**，它们继承于ForkJoinTask. 相关关系如下图所示：

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/07/31/16277429012380.jpg)

解读ForkJoinPool的源码并不容易，虽然它的思想较为简单，但在实现上要考虑的显然更多，加上部分代码可读性一般，所以讲解它的全部源码是不现实的，当然也是没必要的。在下文中，我们将主要介绍其核心的**任务提交和执行**相关的部分源码，其他源码有兴趣的可以自行阅读。

### 1. 构造ForkJoinPool的几种不同方式

ForkJoinPool中有四个核心参数，用于控制线程池的**并行数**、**工作线程的创建**、**异常处理**和**模式指定**等。各参数解释如下：

* `int parallelism`：指定并行级别（parallelism level）。ForkJoinPool将根据这个设定，决定工作线程的数量。如果未设置的话，将使用`Runtime.getRuntime().availableProcessors()`来设置并行级别；
* `ForkJoinWorkerThreadFactory factory`：ForkJoinPool在创建线程时，会通过factory来创建。注意，这里需要实现的是ForkJoinWorkerThreadFactory，而不是ThreadFactory. 如果你不指定factory，那么将由默认的DefaultForkJoinWorkerThreadFactory负责线程的创建工作；
* `UncaughtExceptionHandler handler`：指定异常处理器，当任务在运行中出错时，将由设定的处理器处理；
* `boolean asyncMode`：从名字上看，你可能会觉得它是**异步模式**设置，但其实是设置队列的工作模式：`asyncMode ? FIFO_QUEUE : LIFO_QUEUE`. 当asyncMode为true时，将使用先进先出队列，而为false时则使用后进先出的模式。

围绕上面的四个核心参数，ForkJoinPool提供了三种构造方式，使用时你可以根据需要选择其中的一种。

**（1）方式一：默认无参构造**

在该构造方式中，你无需设定任何参数。ForkJoinPool将根据当前处理器数量来设置并行数量，并使用默认的线程构造工厂。**不推荐**。

```java
 public ForkJoinPool() {
        this(Math.min(MAX_CAP, Runtime.getRuntime().availableProcessors()),
             defaultForkJoinWorkerThreadFactory, null, false);
 }
```

**（2）方式二：通过并行数构造**

在该构造方式中，你可以指定并行数量，以更有效地平衡处理器数量和负载。**建议在设置时，并行级别应低于当前处理器的数量**。

```java
 public ForkJoinPool(int parallelism) {
        this(parallelism, defaultForkJoinWorkerThreadFactory, null, false);
 }
```

**（2）方式三：自定义全部参数构造**

以上两种构造方式都是基于这种构造，它允许你配置所有的核心参数。为了更有效地管理ForkJoinPool，**建议你使用这种构造方式**。

```java
public ForkJoinPool(int parallelism,
                        ForkJoinWorkerThreadFactory factory,
                        UncaughtExceptionHandler handler,
                        boolean asyncMode) {
        this(checkParallelism(parallelism),
             checkFactory(factory),
             handler,
             asyncMode ? FIFO_QUEUE : LIFO_QUEUE,
             "ForkJoinPool-" + nextPoolId() + "-worker-");
        checkPermission();
 }
```

### 2. 按类型提交不同任务

**任务提交**是ForkJoinPool的核心能力之一，在提交任务时你有三种选择，如下面表格所示：

||从非fork/join线程调用|从fork/join调用|
|---|---|---|
|提交异步执行|execute(ForkJoinTask)|ForkJoinTask.fork()|
|等待并获取结果|invoke(ForkJoinTask)|ForkJoinTask.invoke()|
|提交执行获取Future结果|submit(ForkJoinTask)|ForkJoinTask.fork() (ForkJoinTasks are Futures)|

**（1）第一类核心方法：invoke**

invoke类型的方法接受ForkJoinTask类型的任务，并在任务执行结束后，返回泛型结果。如果提交的任务是null，将抛出空指针异常。

```java
 public <T> T invoke(ForkJoinTask<T> task) {
        if (task == null)
            throw new NullPointerException();
        externalPush(task);
        return task.join();
 }
```

**（2）第二类核心方法：execute**

execute类型的方法在提交任务后，不会返回结果。另外要注意的是，ForkJoinPool不仅允许提交ForkJoinTask类型任务，还允许提交**Callable**或**Runnable**任务，因此你可以像使用现有Executors一样使用ForkJoinPool。

当然，Callable或Runnable类型任务时，将会转换为ForkJoinTask类型，具体可以查看任务提交的相关源码。那么，这类任务和直接提交ForkJoinTask任务有什么区别呢？还是有的。**区别在于，由于任务是不可切分的，所以这类任务无法获得任务拆分这方面的效益，不过仍然可以获得任务窃取带来的好处和性能提升**。

```java
 public void execute(ForkJoinTask<?> task) {
        if (task == null)
            throw new NullPointerException();
        externalPush(task);
 }

public void execute(Runnable task) {
        if (task == null)
            throw new NullPointerException();
        ForkJoinTask<?> job;
        if (task instanceof ForkJoinTask<?>) // avoid re-wrap
            job = (ForkJoinTask<?>) task;
        else
            job = new ForkJoinTask.RunnableExecuteAction(task);
        externalPush(job);
 }
```

**（3）第三类核心方法：submit**

submit类型的方法支持三种类型的任务提交：ForkJoinTask类型、Callable类型和Runnable类型。在提交任务后，将返回ForkJoinTask类型的结果。如果提交的任务是null，将抛出空指针异常，并且当任务不能按计划执行的话，将抛出任务拒绝异常。

```java
   public < T > ForkJoinTask < T > submit(ForkJoinTask < T > task) {
       if (task == null)
           throw new NullPointerException();
       externalPush(task);
       return task;
   }

   public < T > ForkJoinTask < T > submit(Callable < T > task) {
       ForkJoinTask < T > job = new ForkJoinTask.AdaptedCallable < T > (task);
       externalPush(job);
       return job;
   }

   public < T > ForkJoinTask < T > submit(Runnable task, T result) {
       ForkJoinTask < T > job = new ForkJoinTask.AdaptedRunnable < T > (task, result);
       externalPush(job);
       return job;
   }

   public ForkJoinTask < ? > submit(Runnable task) {
       if (task == null)
           throw new NullPointerException();
       ForkJoinTask < ? > job;
       if (task instanceof ForkJoinTask < ? > ) // avoid re-wrap
           job = (ForkJoinTask < ? > ) task;
       else
           job = new ForkJoinTask.AdaptedRunnableAction(task);
       externalPush(job);
       return job;
   }

```

### 3. ForkJoinTask

**ForkJoinTask是ForkJoinPool的核心之一，它是任务的实际载体，定义了任务执行时的具体逻辑和拆分逻辑**，本文前面的示例代码就是通过继承它实现。作为一个抽象类，ForkJoinTask的行为有点类似于线程，但它更为轻量，因为它不维护自己的运行时堆栈或程序计数器等。

在类的设计上，ForkJoinTask继承了Future接口，所以也可以将其看作是轻量级的Future，它们之间的关系如下图所示。

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/08/02/16278936733928.jpg)

#### （1）fork与join

`fork()`/`join()`是ForkJoinTask甚至是ForkJoinPool的核心方法，承载着主要的任务协调作用，一个用于任务提交，一个用于结果获取。

**fork-提交任务**

`fork()`方法用于向**当前任务所运行的线程池**中提交任务，比如上文示例代码中的`subTask1.fork()`. 注意，不同于其他线程池的写法，任务提交由任务自己通过调用`fork()`完成，对此不要感觉诧异，`fork()`内部会将任务与当前线程进行关联。

从源码中看，如果当前线程是ForkJoinWorkerThread类型，将会放入该线程的任务队列，否则放入common线程池的任务队列中。**关于common线程池，后续会有介绍**。

```java
    public final ForkJoinTask<V> fork() {
        Thread t;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            ((ForkJoinWorkerThread)t).workQueue.push(this);
        else
            ForkJoinPool.common.externalPush(this);
        return this;
    }
```

**join-获取任务执行结果**

前面，你已经知道可以通过`fork()`提交任务。那么现在，你则可以通过`join()`方法获取任务的执行结果。

调用`join()`时，将阻塞当前线程直到对应的子任务完成运行并返回结果。从源码看，`join()`的核心逻辑由`doJoin()`负责。`doJoin()`虽然很短，但可读性较差，阅读时稍微忍一下。

```java
public final V join() {
    int s;
    // 如果调用doJoin返回的非NORMAL状态，将报告异常
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        reportException(s);
    // 正常执行结束，返回原始结果
    return getRawResult();
}

private int doJoin() {
    int s;
    Thread t;
    ForkJoinWorkerThread wt;
    ForkJoinPool.WorkQueue w;
    //如果已完成，返回状态
    return (s = status) < 0 ? s :
    　//如果未完成且当前线程是ForkJoinWorkerThread，则从该线程中取出workQueue，并尝试将当前task取出执行。如果执行的结果是完成，则返回状态；否则，使用当前线程池awaitJoin方法进行等待
        ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
        (w = (wt = (ForkJoinWorkerThread) t).workQueue).
    tryUnpush(this) && (s = doExec()) < 0 ? s :
        wt.pool.awaitJoin(w, this, 0 L):
     //当前线程非ForkJoinWorkerThread，调用externalAwaitDone方法等待
        externalAwaitDone();
}

final int doExec() {
    int s;
    boolean completed;
    if ((s = status) >= 0) {
        try {
            completed = exec();
        } catch (Throwable rex) {
            return setExceptionalCompletion(rex);
        }
        // 执行完成后，将状态设置为NORMAL
        if (completed)
            s = setCompletion(NORMAL);
    }
    return s;
}
```
#### （2）RecursiveAction与RecursiveTask

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/08/02/16278936733928.jpg)

在ForkJoinPool中，常用的有两种任务类型：**返回结果**的和**不返回结果**的，这方面和ThreadPoolExecutor等线程池是一致的，对应的两个类分别是：**RecursiveAction**和**RecursiveTask**.  从类图中可以看到，它们均继承于ForkJoinTask. 

**RecursiveAction：无结果返回**

RecursiveAction用于递归执行但不需要返回结果的任务，比如下面的排序就是它的典型应用场景。在使用RecursiveAction时，你需要继承并实现它的核心方法`compute()`.

```java
static class SortTask extends RecursiveAction {
    final long[] array;
    final int lo, hi;
    SortTask(long[] array, int lo, int hi) {
        this.array = array;
        this.lo = lo;
        this.hi = hi;
    }
    SortTask(long[] array) {
        this(array, 0, array.length);
    }
    // 核心计算方法
    protected void compute() {
        if (hi - lo < THRESHOLD)
            // 直接执行
            sortSequentially(lo, hi);
        else {
            // 拆分任务
            int mid = (lo + hi) >>> 1;
            invokeAll(new SortTask(array, lo, mid),
                new SortTask(array, mid, hi));
            merge(lo, mid, hi);
        }
    }
    // implementation details follow:
    static final int THRESHOLD = 1000;
    void sortSequentially(int lo, int hi) {
        Arrays.sort(array, lo, hi);
    }
    void merge(int lo, int mid, int hi) {
        long[] buf = Arrays.copyOfRange(array, lo, mid);
        for (int i = 0, j = lo, k = mid; i < buf.length; j++)
            array[j] = (k == hi || buf[i] < array[k]) ?
            buf[i++] : array[k++];
    }
}
```

**RecursiveTask：返回结果**

RecursiveTask用于递归执行需要返回结果的任务，比如前面示例代码中的求和或下面这段求斐波拉契数列求和都是它的典型应用场景。在使用RecursiveTask时，你也需要继承并实现它的核心方法`compute()`.

```java
 class Fibonacci extends RecursiveTask<Integer> {
   final int n;
   Fibonacci(int n) { this.n = n; }
   Integer compute() {
     if (n <= 1)
       return n;
     Fibonacci f1 = new Fibonacci(n - 1);
     f1.fork();
     Fibonacci f2 = new Fibonacci(n - 2);
     return f2.compute() + f1.join();
   }
 }
```

#### （3）ForkJoinTask使用限制

虽然在某些场景下，ForkJoinTask可以通过任务拆解的方式提高执行效率，但是需要注意的是它并非适合所有的场景。**ForkJoinTask在使用时需要谨记一些限制，违背这些限制可能会适得其反甚至引来灾难**。

**为什么这么说呢？**

这是因为，ForkJoinTask最适合用于纯粹的**计算任务**，也就是**纯函数计算**，计算过程中的对象都是独立的，对外部没有依赖。你可以想象，如果大量的任务或被拆分的子任务之间彼此依赖或对外部存在严重阻塞依赖，那将是怎样的画面...用千丝万缕来形容也不为过，外部依赖会带来**任务执行**和**问题排查**方面的双重不确定性。

所以，在理想情况下，提交到ForkJoinPool中的任务**应避免执行阻塞I/O**，以免出现不可控的意外情况。当然，这也并非是绝对的，在必要时你也可以定义和使用可阻塞的ForkJoinTask，只不过你需要付出更多的代价和考虑，使用时应当慎之又慎，本文对此不作叙述。

### 4. 工作队列与任务窃取

前面已经说到，ForkJoinPool与ThreadPoolExecutor有个很大的不同之处在于，ForkJoinPool存在引入了**任务窃取**设计，它是其性能保证的关键之一。

关于任务窃取，简单来说，**就是允许空闲线程从繁忙线程的双端队列中窃取任务**。默认情况下，工作线程从它自己的双端队列的**头部**获取任务。但是，当自己的任务为空时，线程会从其他繁忙线程双端队列的**尾部**中获取任务。这种方法，最大限度地减少了线程竞争任务的可能性。

ForkJoinPool的大部分操作都发生在**工作窃取队列（work-stealing queues ）** 中，该队列由内部类WorkQueue实现。其实，这个队列也不是什么神奇之物，它是Deques的特殊形式，但仅支持三种操作方式：**push**、**pop**和**poll**（也称为窃取）。当然，在ForkJoinPool中，队列的读取有着严格的约束，`push`和`pop`仅能从其所属线程调用，而`poll`则可以从其他线程调用。换句话说，前两个方法是留给自己用的，而第三种方法则是为了方便别人来窃取任务用的。**任务窃取的相关过程，可以用下面这幅图来表示，这幅图建议你收藏**：

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/07/31/16277428621515.jpg)

看到这里，不知你是否会有疑问：**为什么工作线程总是从自己的头部获取任务？为什么要这样设计？首先处理队列中等待时间较长的任务难道不是更有意义吗**？

答案当然不会是“**更有意义**”。**这样做的主要原因是为了提高性能，通过始终选择最近提交的任务，可以增加资源仍分配在CPU缓存中的机会，这样CPU处理起来要快一些**。而窃取者之所以从尾部获取任务，**则是为了降低线程之间的竞争可能，毕竟大家都从一个部分拿任务，竞争的可能要大很多**。

此外，这样的设计还有一种考虑。**由于任务是可分割的，那队列中较旧的任务最有可能粒度较大，因为它们可能还没有被分割，而空闲的线程则相对更有“精力”来完成这些粒度较大的任务**。

### 5. ForkJoinPool监控

对于一个复杂框架来说，实时地了解ForkJoinPool的内部状态是十分必要的。因此，ForkJoinPool提供了一些常用方法。通过这些方法，你可以了解当前的工作线程、任务处理等情况。

**（1）获取运行状态的线程总数**

```java
public int getRunningThreadCount() {
    int rc = 0;
    WorkQueue[] ws;
    WorkQueue w;
    if ((ws = workQueues) != null) {
        for (int i = 1; i < ws.length; i += 2) {
            if ((w = ws[i]) != null && w.isApparentlyUnblocked())
                ++rc;
        }
    }
    return rc;
}
```
**（2）获取活跃线程数量**
```java
public int getActiveThreadCount() {
        int r = (config & SMASK) + (int)(ctl >> AC_SHIFT);
        return (r <= 0) ? 0 : r; // suppress momentarily negative values
}

```
**（3）判断ForkJoinPool是否空闲**
```java
public boolean isQuiescent() {
    return (config & SMASK) + (int)(ctl >> AC_SHIFT) <= 0;
}
```
**（4）获取任务窃取数量**

```java
public long getStealCount() {
    AtomicLong sc = stealCounter;
    long count = (sc == null) ? 0 L : sc.get();
    WorkQueue[] ws;
    WorkQueue w;
    if ((ws = workQueues) != null) {
        for (int i = 1; i < ws.length; i += 2) {
            if ((w = ws[i]) != null)
                count += w.nsteals;
        }
    }
    return count;
}
```
**（5）获取队列中的任务数量**
```java
public long getQueuedTaskCount() {
    long count = 0;
    WorkQueue[] ws;
    WorkQueue w;
    if ((ws = workQueues) != null) {
        for (int i = 1; i < ws.length; i += 2) {
            if ((w = ws[i]) != null)
                count += w.queueSize();
        }
    }
    return count;
}

```
**（6）获取已提交的任务数量**
```java
public int getQueuedSubmissionCount() {
    int count = 0;
    WorkQueue[] ws;
    WorkQueue w;
    if ((ws = workQueues) != null) {
        for (int i = 0; i < ws.length; i += 2) {
            if ((w = ws[i]) != null)
                count += w.queueSize();
        }
    }
    return count;
}
```

## 四、警惕ForkJoinPool#commonPool

在上文中所示的源码中，你可能已经在多处注意到commonPool的存在。在ForkJoinPool中，commonPool是一个**共享的、静态的**线程池，并且在实际使用时才会进行懒加载，**Java8中的CompletableFuture和并行流（Parallel Streams）用的就是它**。不过，**使用CompletableFuture时你可以指定自己的线程池，但是并行流在使用时却不可以，这也是我们要警惕的地方**。为什么这么说呢？

ForkJoinPool中的commonPool设计初衷是为了降低线程池的重复创建，让一些任务共用同一个线程池，毕竟创建线程池和创建线程都是昂贵的。然而，凡事都有两面性，commonPool在某些场景下确实可以达到线程池复用的目的，但是，**如果你决定与别人分享自己空间，那么当你想使用它的时候，它可能不再完全属于你**。也就是说，当你想用commonPool时，它可能已经其他任务填满了。

提交到ForkJoinPool中的任务一般有两类：**计算类型**和**阻塞类型**。考虑一个场景，应用中多处都在使用这个共享线程池，有人在某处做了个不当操作，比如往池子里丢入了阻塞型任务，那么结果会怎样？结果当然是，**整个线程池都有可能被阻塞**！如此，**整个应用都面临着被拖垮的风险**。看到这里，对于Java8中的并行流的使用，你就应该高度警惕了。
 
那怎么避免这种情况发生呢？答案是尽量避免使用commonPool，并且在需要运行阻塞任务时，应当创建独立的线程池，和系统的其他部分保持隔离，以免风险扩散。

## 五、ForkJoinPool性能评估

为了测试ForkJoinPool的性能，我做了一组**简单的**、**非正式**实验。实验分三组进行，为了尽可能让每组的数据客观，每组实验均运行5次，取最后的平均数。

* **实验代码**：本文第一部分的示例代码；
* **实验环境**：Mac；
* **JDK版本**：8；
* **任务分隔阈值**：100

**实验结果如下方表格所示：**

<table style="width:100%">
   <thead>
      <tr>
         <th rowspan="2">实验次数</th>
         <th colspan="2">1000量级耗时（毫秒）</th>
         <th colspan="2">1000000量级耗时（毫秒）</th>
         <th colspan="2">1000000000量级耗时（毫秒）</th>
      </tr>
      <tr>
         <th>Fork/Join</th>
         <th>单线程</th>
         <th>Fork/Join</th>
         <th>单线程</th>
         <th>Fork/Join</th>
         <th>单线程</th>
      </tr>
   </thead>
   <tbody>
      <tr>
         <td>1</td>
         <td>4</td>
         <td>0</td>
         <td>34</td>
         <td>5</td>
         <td>1157</td>
         <td>313</td>
      </tr>
      <tr>
         <td>2</td>
         <td>3</td>
         <td>0</td>
         <td>34</td>
         <td>6</td>
         <td>848</td>
         <td>344</td>
      </tr>
      <tr>
         <td>3</td>
         <td>5</td>
         <td>0</td>
         <td>16</td>
         <td>9</td>
         <td>1069</td>
         <td>325</td>
      </tr>
      <tr>
         <td>4</td>
         <td>4</td>
         <td>0</td>
         <td>35</td>
         <td>8</td>
         <td>955</td>
         <td>307</td>
      </tr>
      <tr>
         <td>5</td>
         <td>5</td>
         <td>0</td>
         <td>30</td>
         <td>22</td>
         <td>922</td>
         <td>385</td>
      </tr>
      <tr>
         <td>平均</td>
         <td>4.2</td>
         <td>0</td>
         <td>29.8</td>
         <td>10</td>
         <td>990.2</td>
         <td>334.8</td>
      </tr>
   </tbody>
</table>

从实验结果（0表示不到1毫秒）来看，**ForkJoinPool的性能竟然不如单线程的效率高**！这样的结果，似乎很惊喜、很意外...**然而，为什么会这样**？

不要惊讶，之所以会出现这个令你匪夷所思的结果，**其原因在于任务拆分的粒度过小**！在上面的测试中，任务拆分阈值仅为100，导致Fork/Join在计算时出现大量的任务拆分动作，也就是任务分的太细，大量的任务拆分和管理也是需要额外成本的。

以0~1000000求和为例，当把阈值从**100**调整为**100000**时，其结果结果如下。可以看到，Fork/Join的优势就体现出来了。

```shell
======
ForkJoin任务拆分：16383
ForkJoin计算结果：499999999500000000
ForkJoin计算耗时：143
======
单线程计算结果：499999999500000000
单线程计算耗时：410
```

那么，问题又来了，哪些因素会影响Fork/Join的性能呢？

根据经验和实验，**任务总数**、**单任务执行耗时**以及**并行数**都会影响到性能。所以，**当你使用Fork/Join框架时，你需要谨慎评估这三个指标，最好能通过模拟对比评估，不要凭感觉冒然在生产环境使用**。

## 小结

以上就是关于ForkJoinPool的全部内容。Fork/Join是一种基于分治算法的模型，在并发处理计算型任务时有着显著的优势。其效率的提升主要得益于两个方面：
* **任务切分**：将大的任务分割成更小粒度的小任务，让更多的线程参与执行；
* **任务窃取**：通过任务窃取，充分地利用空闲线程，并减少竞争。

在使用ForkJoinPool时，需要特别注意任务的类型是否为**纯函数计算类型**，也就是这些任务不应该关心状态或者外界的变化，这样才是最安全的做法。如果是阻塞类型任务，那么你需要谨慎评估技术方案。虽然ForkJoinPool也能处理阻塞类型任务，但可能会带来复杂的管理成本。

而在性能方面，要认识到Fork/Join的性能并不是开箱即来，而是需要你去评估和验证一些重要指标，通过数据对比得出最佳结论。

此外，ForkJoinPool虽然提供了commonPool，但出于潜在的风险考虑，不推荐使用或谨慎使用。

正文到此结束，恭喜你又上了一颗星✨

## 夫子的试炼

* 动手：使用ForkJoinPool实现List数组排序。

## 延伸阅读与参考资料

* A Java Fork/Join Framework（Doug Lea）：http://gee.cs.oswego.edu/dl/papers/fj.pdf
* Java Fork and Join using ForkJoinPool：http://tutorials.jenkov.com/java-util-concurrent/java-fork-and-join-forkjoinpool.html
* Introduction to the Fork/Join Framework：https://www.pluralsight.com/guides/introduction-to-the-fork-join-framework
* The Unfairly Unknown ForkJoinPool：https://medium.com/swlh/the-unfairly-unknown-forkjoinpool-c262777def6a
* A Java? Fork-Join Calamity：http://coopsoft.com/ar/CalamityArticle.html
* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。