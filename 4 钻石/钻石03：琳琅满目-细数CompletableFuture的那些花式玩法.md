欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第26篇**，砖石中的**第3篇**。

从Java8开始，JDK引入了很多新的特性，包括lambda表达式、流式计算等，以及本文所要详述的**CompletableFuture**. 关于CompletableFuture，你可能首先会联想到**Future**接口，对于它我们并不陌生，在ThreadPoolExecutor和ForkJoinPool中都见过它的身影。如果你对此感到困惑的话，不妨先阅读我们的前两篇文章。

Future的接口定义本身并不复杂，使用起来也较为简单，它的核心是`get()`和`isDone()`方法。然而，**Future的简单也导致了它在某些方面会存在先天性的不足**。在某些场景下，Future可能无法满足我们的需求，比如我们无法通过Future实现对并发任务的编排。不过，幸运的是，本文所要介绍的CompletableFuture弥补了Future多方面的不足之处，它可能成为你的最佳之选，这也是本文为什么要谈CompletableFuture的原因。

在这篇文章中，我们将结合Future和线程池，探讨CompletableFuture与Future的不同之处，以及它的核心能力和最佳实践。

## 一、理解CompletableFuture

### 1. Future的局限性

从本质上说，**Future表示一个异步计算的结果**。它提供了`isDone()`来检测计算是否已经完成，并且在计算结束后，可以通过`get()`方法来获取计算结果。在异步计算中，Future确实是个非常优秀的接口。但是，它的本身也确实存在着许多限制：

* **并发执行多任务**：Future只提供了get()方法来获取结果，并且是阻塞的。所以，除了等待你别无他法；
* **无法对多个任务进行链式调用**：如果你希望在计算任务完成后执行特定动作，比如发邮件，但Future却没有提供这样的能力；
* **无法组合多个任务**：如果你运行了10个任务，并期望在它们全部执行结束后执行特定动作，那么在Future中这是无能为力的；
* **没有异常处理**：Future接口中没有关于异常处理的方法；

### 2. CompletableFuture与Future的不同

**简单地说，CompletableFuture是Future接口的扩展和增强**。CompletableFuture完整地继承了Future接口，并在此基础上进行了丰富地扩展，完美地弥补了Future上述的种种问题。更为重要的是，CompletableFuture**实现了对任务的编排能力**。借助这项能力，我们可以轻松地组织不同任务的运行顺序、规则以及方式。**从某种程度上说，这项能力是它的核心能力**。而在以往，虽然通过CountDownLatch等工具类也可以实现任务的编排，但需要复杂的逻辑处理，不仅耗费精力且难以维护。

### 3. CompletableFuture初体验

当然，百闻不如一见，既然CompletableFuture如此神乎其神，我们不妨通过一个特定的场景来体验CompletableFuture的用法。

众所周知，王者中有注明的“**草丛三杰（B）**”，妲己就是其中之一，她蹲草丛的本领可谓一绝。话说这天，妲己远远看见敌方小鲁班蹦蹦跳跳地走来，对付这样的脆皮最适合在草丛中来一波操作。于是，妲己侧身躲进了草丛，在小鲁班欢快地路过时，妲己一套熟练的**231连招**秒杀了小鲁班。小鲁班死不瞑目，因为他甚至还没看清对手的模样，很快啊！

在这个过程中，包含几组动作：捉拿鲁班、打出技能2、打出技能3以及打出技能1. 我们可以通过**CompletableFuture**的链式调用来表达这些动作：

```java
CompletableFuture.supplyAsync(CompletableFutureDemo::活捉鲁班)
    .thenAccept(player -> note(player.getName())) // 接收supplyAsync的结果，获得对方名字
    .thenRun(() -> attack("2技能-偶像魅力：鲁班受到妲己285点法术伤害，并眩晕1.5秒..."))
    .thenRun(() -> attack("3技能-女王崇拜：妲己放出5团狐火，鲁班受到325++点法术伤害..."))
    .thenRun(() -> attack("1技能-灵魂冲击：鲁班受到妲己520点点法术伤害..."))
    .thenRunAsync(() -> note("鲁班，卒...")); // 使用线程池的其他线程
```
你看，使用CompletableFuture编排动作是不是很容易？在这段只有6行的代码中，我们已经使用了supplyAsync()和thenAccept()等4中不同的方法，并且同时使用了同步和异步。在以往，如果手工实现的话，至少需要洋洋洒洒几十行代码。那CompletableFuture是如何做到如此神功的呢？接着往下看。


## 二、CompletableFuture的核心设计

总体而言，CompletableFuture实现了**Future**和**CompletionStage**两个接口，并且只有少量的属性。但是，它有近2400余行的代码，并且关系复杂。所以，在核心设计方面，我们不会展开讨论。

现在，你已经知道，Future接口仅提供了`get()`和`isDone`这样的简单方法，仅凭Future无法为CompletableFuture提供丰富的能力。那么，CompletableFuture又是如何扩展自己的能力的呢？这就不得不说**CompletionStage**接口了，它是CompletableFuture核心，也是我们要关注的重点。

顾名思义，根据CompletionStage名字中的“**Stage**”，你可以把它理解为任务编排中的**步骤**。所谓步骤，即任务编排的基本单元，它可以是**一次纯粹的计算**或者是**一个特定的动作**。在一次编排中，会包含多个步骤，这些步骤之间会存在**依赖**、**链式**和**组合**等不同的关系，也存在**并行**和**串行**的关系。这种关系，类似于Pipeline或者流式计算。

既然是编排，就需要维护任务的创建、建立计算关系。为此，CompletableFuture提供了多达**50**多个方法，**在数量上确实庞大且令人瞠目结舌**，想要全部理解显然不太可能，当然也没有必要。虽然CompletableFuture的方法数量众多，但是在理解时仍有规律可循，我们**可以通过分类的方式简化对方法的理解**，理解了**类型**和**变种**，基本上我们也就掌握了CompletableFuture的核心能力。

根据类型，这些方法可以总结为以下**四类**，其他大部分方法都是基于这四种类型的变种：

|类型|接收参数|返回结果|支持异步|
|---|---|---|---|
|`Supply`|✘|✔︎|✔︎|
|`Apply`|✔︎|✔︎|✔︎|
|`Accept`|✔︎|✘|✔︎|
|`Run`|✘|✘|✔︎|✔︎|

**关于方法的变种**

上述接种类型的方法一般都有三个变种方法：**同步**、**异步**和**指定线程池**。比如， `thenApply()`的三个变种方法如下所示：

```java
<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```

下面这幅类图，展示了CompletableFuture和Future、CompletionStage以及Completion之间的关系。当然，由于方法众多，这幅图中并没有全部呈现，而是仅选取了部分重要的方法。

![](https://writting.oss-cn-beijing.aliyuncs.com/2021/08/10/16285808689234.jpg)


## 三、CompletableFuture的核心用法

前面已经说过，CompletableFuture的核心方法总共分为四类，而这四类方法又分为两种模式：**同步和异步**。所以，我们从这四类方法中选取了部分核心的API，它们都是我们经常用到的API。

* **同步**：使用当前线程运行任务；
* **异步**：使用CompletableFuture线程池其他线程运行任务，异步方法的名字中带有`Async`.

### 1. runAsync

`runAsync()`是CompletableFuture最常用的方法之一，它可以接收一个待运行的任务并返回一个CompletableFuture . 

当我们想异步运行某个任务时，在以往需要手动实现Thread或者借助Executor实现。而通过runAsync()`就简单多了。比如，我们可以直接传入**Runnable**类型的任务：

```java
CompletableFuture.runAsync(new Runnable() {
    @Override
    public void run() {
        note("妲己进入草丛蹲点...等待小鲁班出现");
    }
});
```

此外，在Java8及之后的JDK版本中，我们还可以使用lambda表达式进一步简化代码：

```java
CompletableFuture.runAsync(() -> note("妲己进入草丛蹲点...等待小鲁班出现"));
```

这样看起来是不是很简单？相信很多同学也是采用这样的方式来使用`runAsync()`.  **不过，如果你也这么用，那么你就要小心了**，这里有陷阱。先卖个关子，文末尾会对CompletableFuture线程池做简要的讲解，帮助你如何避免采坑。

### 2. supply与supplyAsync

对于`supply()`这个方法，很多人第一印象可能会比较懵，不知道它是做什么的。但其实，它的名字已经说明了一切：所谓“**supply**”当然是提供结果的！换句话说，当我们使用`supply()`时，就**表明我们会返回一个结果，并且这个结果可以被后续的任务所使用**。

举个例子，在下面的示例代码中，我们通过`supplyAsync()`返回了结果，而这个结果在后续的`thenApply()`被使用。

```java
// 创建nameFuture，返回姓名
CompletableFuture <String> nameFuture = CompletableFuture.supplyAsync(() -> {
    return "妲己";
});

 // 使用thenApply()接收nameFuture的结果，并执行回调动作
CompletableFuture <String> sayLoveFuture = nameFuture.thenApply(name -> {
    return "爱你，" + name;
});

//阻塞获得表白的结果
System.out.println(sayLoveFuture.get()); // 爱你，妲己
```

你看，一旦理解了`supply()`的含义，它也就如此简单了。如果你希望用新的线程运行任务，可以使用`supplyAsync()`. 

### 3. thenApply与thenApplyAsync

刚才，在前面我们已经介绍了`supply()`，已经知道它是用于提供结果的，并且顺带提了`thenApply()`. 很明显，不用说你可能已经知道`thenApply()`是`supply()`的搭档，用于接收`supply()`的执行结果，并执行特定的代码逻辑，最后返回CompletableFuture结果。

```java

 // 使用thenApply()接收nameFuture的结果，并执行回调动作
CompletableFuture <String> sayLoveFuture = nameFuture.thenApply(name -> {
    return "爱你，" + name;
});

public <U> CompletableFuture <U> thenApplyAsync(
    Function <? super T, ? extends U> fn) {
    return uniApplyStage(null, fn);
}
```

### 4. thenAccept与thenAcceptAsync

作为`supply()`的档案，`thenApply()`并不是唯一的存在，`thenAccept()`也是。但与`thenApply()`不同，`thenAccept()`**只接收数据，但不会返回，它的返回类型是Void**.

```java

CompletableFuture<Void> sayLoveFuture = nameFuture.thenAccept(name -> {
     System.out.println("爱你，" + name);
});
        
public CompletableFuture < Void > thenAccept(Consumer < ? super T > action) {
    return uniAcceptStage(null, action);
}
```

### 5. thenRun

`thenRun()`就比较简单了，**不接收任务的结果，只运行特定的任务，并且也不返回结果**。

 ```java
public CompletableFuture < Void > thenRun(Runnable action) {
    return uniRunStage(null, action);
}
```

所以，如果你在回调中不想返回任何的结果，只运行特定的逻辑，那么你可以考虑使用`thenAccept`和`thenRun`. 一般来说，这两个方法会在调用链的最后面使用。.

### 6. thenCompose与 thenCombine

以上几种方法都是各玩各的，但`thenCompose()`与`thenCombine()`就不同了，它们可以实现对**依赖**和**非依赖**两种类型的任务的编排。

**编排两个存在依赖关系的任务**

在前面的例子中，在接收前面任务的结果时，我们使用的是thenApply().  也就是说，sayLoveFuture在执行时必须依赖nameFuture的完成，否则执行个锤子。

```java
// 创建Future
CompletableFuture <String> nameFuture = CompletableFuture.supplyAsync(() -> {
    return "妲己";
});

 // 使用thenApply()接收nameFuture的结果，并执行回调动作
CompletableFuture <String> sayLoveFuture = nameFuture.thenApply(name -> {
    return "爱你，" + name;
});
```

但其实，除了thenApply()之外，我们还可以使用`thenCompose()`来编排两个存在依赖关系的任务。比如，上面的示例代码可以写成：

```java
// 创建Future
CompletableFuture <String> nameFuture = CompletableFuture.supplyAsync(() -> {
    return "妲己";
});

CompletableFuture<String> sayLoveFuture2 = nameFuture.thenCompose(name -> {
    return CompletableFuture.supplyAsync(() -> "爱你，" + name);
});
```
可以看到，`thenCompose()`和`thenApply()`的核心不同之处**在于它们的返回值类型**：

* `thenApply()`：返回计算结果的原始类型，比如返回String;
* `thenCompose()`：返回CompletableFuture类型，比如返回CompletableFuture<String>. 

**组合两个相互独立的任务**

考虑一个场景，当我们在执行某个任务时，需要其他任务就绪才可以，应该怎么做？这样的场景并不少见，我们可以使用前面学过的并发工具类实现，也可以使用`thenCombine()`实现。

举个例子，当我们计算某个英雄（比如妲己）的胜率时，我们需要获取她参与的**总场次（rounds**），以及她**获胜的场次（winRounds）**，然后再通过`winRounds / rounds`来计算。对于这个计算，我们可以这么做：

```java
CompletableFuture < Integer > roundsFuture = CompletableFuture.supplyAsync(() -> 500);
CompletableFuture < Integer > winRoundsFuture = CompletableFuture.supplyAsync(() -> 365);

CompletableFuture < Object > winRateFuture = roundsFuture
    .thenCombine(winRoundsFuture, (rounds, winRounds) -> {
        if (rounds == 0) {
            return 0.0;
        }
        DecimalFormat df = new DecimalFormat("0.00");
        return df.format((float) winRounds / rounds);
    });
System.out.println(winRateFuture.get());

```

`thenCombine()`将另外两个任务的结果同时作为参数，参与到自己的计算逻辑中。在另外两个参数未就绪时，它将会处于等待状态。

### 7. allOf与anyOf

`allOf()`与`anyOf()`也是一对孪生兄弟，当我们需要对**多个Future**的运行进行组织时，就可以考虑使用它们：

* `allOf()`：给定一组任务，等待**所有任务**执行结束；
* `anyOf()`：给定一组任务，等待**其中任一**任务执行结束。

`allOf()`与`anyOf()`的方法签名如下：
```java
static CompletableFuture<Void>	 allOf(CompletableFuture<?>... cfs)
static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
```
需要注意的是，`anyOf()`将返回完任务的执行结果，但是`allOf()`不会返回任何结果，它的返回值是**Void**. 

`allOf()`与`anyOf()`的示例代码如下所示。我们创建了roundsFuture和winRoundsFuture，并通过`sleep`模拟它们的执行时间。在执行时，winRoundsFuture将会先返回结果，所以当我们调用 CompletableFuture.anyOf时也会发现输出的是**365**. 

```java
 CompletableFuture < Integer > roundsFuture = CompletableFuture.supplyAsync(() -> {
   try {
     Thread.sleep(200);
     return 500;
   } catch (InterruptedException e) {
     return null;
   }
 });
 CompletableFuture < Integer > winRoundsFuture = CompletableFuture.supplyAsync(() -> {
   try {
     Thread.sleep(100);
     return 365;
   } catch (InterruptedException e) {
     return null;
   }
 });

 CompletableFuture < Object > completedFuture = CompletableFuture.anyOf(winRoundsFuture, roundsFuture);
 System.out.println(completedFuture.get()); // 返回365

 CompletableFuture < Void > completedFutures = CompletableFuture.allOf(winRoundsFuture, roundsFuture);
```

在CompletableFuture之前，如果要实现所有任务结束后执行特定的动作，我们可以考虑CountDownLatch等工具类。现在，则多了一选项，我们也可以考虑使用`CompletableFuture.allOf`. 

## 四、CompletableFuture中的异常处理

对于任何框架来说，对异常的处理都是必不可少的，CompletableFuture当然也不会例外。前面，我们已经了解了CompletableFuture的核心方法。现在，我们再来看如何处理计算过程中的异常。

考虑下面的情况，当`rounds=0`时，将抛出运行时异常。此时，我们应该如何处理？
```java
CompletableFuture < ? extends Serializable > winRateFuture = roundsFuture
    .thenCombine(winRoundsFuture, (rounds, winRounds) -> {
        if (rounds == 0) {
            throw new RuntimeException("总场次错误");
        }
        DecimalFormat df = new DecimalFormat("0.00");
        return df.format((float) winRounds / rounds);
    });
System.out.println(winRateFuture.get());
```

在CompletableFuture链式调用中，**如果某个任务发生了异常，那么后续的任务将都不会再执行**。对于异常，我们有两种处理方式：`exceptionally()`和`handle()`. 

### 1. 使用exceptionally()回调处理异常

在链式调用的尾部使用`exceptionally()`，捕获异常并返回错误情况下的默认值。需要注意的是，`exceptionally()`**仅在发生异常时才会调用**。

```java
CompletableFuture < ? extends Serializable > winRateFuture = roundsFuture
    .thenCombine(winRoundsFuture, (rounds, winRounds) -> {
        if (rounds == 0) {
            throw new RuntimeException("总场次错误");
        }
        DecimalFormat df = new DecimalFormat("0.00");
        return df.format((float) winRounds / rounds);
    }).exceptionally(ex -> {
        System.out.println("出错：" + ex.getMessage());
        return "";
    });
System.out.println(winRateFuture.get());
```

### 2. 使用handle()处理异常

除了`exceptionally()`，CompletableFuture也提供了`handle()`来处理异常。不过，与`exceptionally()`不同的是，当我们在调用链中使用了`handle()`，**那么无论是否发生异常，都会调用它**。所以，在`handle()`方法的内部，我们需要通过 `if (ex != null) `来判断是否发生了异常。

```java
CompletableFuture < ? extends Serializable > winRateFuture = roundsFuture
    .thenCombine(winRoundsFuture, (rounds, winRounds) -> {
        if (rounds == 0) {
            throw new RuntimeException("总场次错误");
        }
        DecimalFormat df = new DecimalFormat("0.00");
        return df.format((float) winRounds / rounds);
    }).handle((res, ex) -> {
        if (ex != null) {
            System.out.println("出错：" + ex.getMessage());
            return "";
        }
        return res;
    });
System.out.println(winRateFuture.get());
```

当然，如果我们允许某个任务发生异常而不中断整个调用链路，那么可以在其内部通过`try-catch`消化掉。

## 五、CompletableFuture中的线程池

在前面我们已经说过CompletableFuture中的任务有**同步**、**异步**和**指定线程池**三个变种。比如，当我们调用`thenAccept()`时，将不会使用新的线程，而是使用当前线程。而当我们使用`thenAcceptAsync()`时，则会创建新的线程。**那么，在前面的所有示例中，我们都从未创建过线程，CompletableFuture又是如何创建新线程的**？

答案是**ForkJoinPool.commonPool()**，我们熟悉的老朋友又回来了，还是它。当需要新的线程时，CompletableFuture会从commonPool中获取线程，相关源码如下：

```java
public static CompletableFuture<Void> runAsync(Runnable runnable) {
    return asyncRunStage(asyncPool, runnable);
}
private static final Executor asyncPool = useCommonPool ? ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```

可问题是，我们已经知道了commonPool的潜在风险，**在生产环境中使用无异于给自己挖坑**。那怎么办呢？当然是**自定义线程池**，如此重要的东西务必要掌握在自己的手上。**换句话说，当我决定使用CompletableFuture的时候，默认就是我们要创建自己的线程池**。不要偷懒，更不要存在侥幸心理。

CompletableFuture中每个核心类型的方法都提供了自定义线程池的重载，使用起来也较为简单：

```java

// supplyAsync中可以指定线程池的方法
public static < U > CompletableFuture < U > supplyAsync(Supplier < U > supplier,
    Executor executor) {
    return asyncSupplyStage(screenExecutor(executor), supplier);
}

// 自定义线程池示例
Executor executor = Executors.newFixedThreadPool(10);

CompletableFuture < Integer > roundsFuture = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(200);
        return 500;
    } catch (InterruptedException e) {
        return null;
    }
}, executor);
```

## 小结

至此，关于CompletableFuture的讲解已经全部结束。我们已经知道，CompletableFuture是Future的扩展和增强，并提供了大量强大且好玩的优秀特性。这些特性可以帮助我们优雅地解决一些场景问题，而在此之前我们要实现相同的方案可能要花费很大的代价。

当然，CompletableFuture这朵玫瑰虽然很漂亮，但它的刺也同样尖锐，它并不是天生完美。因此，在使用CompletableFuture时仍要遵循一些必要的约束：

* **自定义线程池**：当你决定在生产环境使用CompletableFuture的时候，你应该同时准备好对应的线程池策略，而不是偷懒地使用默认的线程池；
* **团队共识**：技术就是这样，好与不好总是会有不同的标准。当你说好的时候，你的队友可能并不这么认为，反之你也可能也会反对某种技术观点。因此，当你决定采用CompletableFuture的时候，最好和团队同步你的策略，让大家都了解它的优点和潜在的风险，各行其是绝对不是好的策略。

最后，CompletableFuture的源码有近**2400**行，并且有大量的API. 说实话，在王者系列所分析的源码文章中，CompletableFuture的源码是截止目前最难以理解的。如果将源码展开讲解的话，大概需要数万字，这将直接劝退百分之九十以上的读者。所以，我们也不建议你硬啃所有的源码，而是**建议在归纳分类的基础上，有针对性地掌握它的重点部分**。当然，如果你饶有兴趣地读完了它所有的源码，在此给你点赞。

正文到此结束，恭喜你又上了一颗星✨

## 夫子的试炼

* 动手：编写代码体验`runAsync()` 的用法，并指定线程池。

## 延伸阅读与参考资料

* https://thepracticaldeveloper.com/differences-between-completablefuture-future-and-streams/#conclusions-pros-and-cons-of-completablefuture

* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。

