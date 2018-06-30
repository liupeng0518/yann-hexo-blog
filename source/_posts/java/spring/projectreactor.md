---
title: 由表及里学 ProjectReactor
date: 2018-06-26 22:49:48
tags:
toc: true
categories: ["java", "spring", "projectreactor"]
---
我们都知道 ProjectReactor 是借鉴 Rxjava 的，这个在Android开发中一个极为重要的库，现在因为后端服务采用Netty，又在Reactive的东风之下，后端也选择采用 [`响应式编程模式`](https://en.wikipedia.org/wiki/Reactive_programming) 进行开发，响应式编程不是一个新概念，响应式编程就是用异步数据流进行编程，在传统的GUI应用中因为不能阻塞 `绘图IO` 所以有很多基于事件的编程模式，响应式编程提高了代码的抽象水平，因此能专注于那些定义业务逻辑的事件的依存关系，而无需摆弄大量的线程相关实现细节。
下文简称 `projectreactor` 为 `PRR`。

<!-- more -->

{% raw %}

<script src="https://lib.baomitu.com/webfont/1.6.28/webfontloader.js"></script>
<script src="https://lib.baomitu.com/snap.svg/0.5.1/snap.svg-min.js"></script>
<script src="https://lib.baomitu.com/underscore.js/1.9.0/underscore-min.js"></script>
<script src="https://lib.baomitu.com/raphael/2.2.7/raphael.min.js"></script>
<script src="https://lib.baomitu.com/js-sequence-diagrams/1.0.6/sequence-diagram-min.js"></script>

{% endraw %}


## 响应式编程
本质上的 `响应式编程模式` 就是一个 `观察者模式`。
![观察者模式](https://s1.ax1x.com/2018/06/29/PFCbwQ.png)

### 为什么需要响应式编程
为什么需要响应式编程，在文档也明确的表示，简而言之就是，我们针对大量的并发用户，我们可以选择
- 并行化（parallelize）:使用更多的线程和硬件资源
- 基于`现有的资源`提高执行效率 

第一种方式，往往采用分布式计算，在这里不做多展开。
第二种方式，通过编写 `异步非阻塞` 的代码，可以减少资源的浪费，在Java中一般采用，`回调（Callbacks）` 和 `Futures` ，但是这两种方式都有局限性，
- 回调很难组合起来，因为很快就会导致代码难以理解和维护（即所谓的“回调地狱（callback hell）”），
- Futures 比回调要好一点，但即使在 Java 8 引入了 CompletableFuture，它对于多个处理的组合仍不够好用。 编排多个 Futures 是可行的，但却不易。此外，Future 还有一个问题：当对 Future 对象最终调用 get() 方法时，仍然会导致阻塞，并且缺乏对多个值以及更进一步对错误的处理。
- [`关于原因的详细阅读`](http://projectreactor.io/docs/core/release/reference/#_asynchronicity_to_the_rescue)


### 官方的例子

```java
userService.getFavorites(userId) ➊
           .flatMap(favoriteService::getDetails)  ➋
           .switchIfEmpty(suggestionService.getSuggestions())  ➌
           .take(5)  ➍
           .publishOn(UiUtils.uiThreadScheduler())  ➎
           .subscribe(uiList::show, UiUtils::errorPopup);  ➏
```
➊ 根据用户ID获得喜欢的信息（打开一个 `Publisher`）  
➋ 使用 `flatMap` 操作获得详情信息  
➌ 使用 `switchIfEmpty` 操作，在没有喜欢数据的情况下，采用系统推荐的方式获得  
➍ 取前五个  
➎ 在 `uiThread` 上进行发布  
➏ 最终的消费行为  

➊➋➌➍ 的行为就看起来很直观，这个和我们使用 [`Java 8 中的 Streams 编程`](http://www.runoob.com/java/java8-streams.html) 极为相近，但是这里实现和 Strem 是不同的，在后续的分析会展开。
➎ 和 ➏ 在我们之前的编码（注：传统后台服务）中没有遇见过类似的，这里的行为，我们可以在后续的 [`Reference#schedulers`](http://projectreactor.io/docs/core/release/reference/#schedulers) 中可以得知，`publishOn` 将影响后续的行为操作所在的线程，那我们就明白了，之前的操作会在某个线程中执行，而最后一个 `subscribe()` 函数将在 `uiThread` 中执行。


如果非常着急话可以先阅读 [**小结图文**](#小结一下)

## SPI 模型定义
### Publisher 即被观察者
Publisher 在 `PRR` 中 所承担的角色也就是传统的 `观察者模式` 中的 `被观察者对象`，在 `PRR` 的定义也极为简单。
```java
package org.reactivestreams;
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
```

`Publisher` 的定义可以看出来，`Publisher` 接受 `Subscriber`，非常简单的一个接口。但是这里有个有趣的小细节，这个类所在的包是 `org.reactivestreams`，这里的做法和传统的 J2EE 标准类似，我们使用标准的 `Javax` 接口定义行为，不定义具体的实现。

### Subscriber 即观察者
Subscriber 在 `PRR` 中 所承担的角色也就是传统的 `观察者模式` 中的 `观察者对象`，在 `PRR` 的定义要多一些。
```java
public interface Subscriber<T> {
    public void onSubscribe(Subscription s); ➊
    public void onNext(T t); ➋
    public void onError(Throwable t); 
    public void onComplete(); ➍
}
```
➊ 订阅时被调用   
➋ 每一个元素接受时被触发一次  
➌ 当在触发错误的时候被调用  
➍ 在接受完最后一个元素最终完成被调用  

`Subscriber` 的定义可以看出来，`Publisher` 是主要的行为对象，用来描述我们最终的执行逻辑。

### Subscription 桥接者
在最基础的 `观察者模式` 中，我们只是需要 `Subscriber 观察者` `Publisher 发布者`，而在 `PRR` 中增加了一个 `Subscription` 作为 `Subscriber` `Publisher` 的桥接者。

```java
public interface Subscription {
    public void request(long n); ➊
    public void cancel(); ➋
}
```
➊ 获取 N 个元素往下传递
➋ 取消执行

为什么需要这个对象，笔者觉得是一是为了解耦合，第二在 `Reference` 中有提到 `Backpressure` 也就是下游可以保护自己不受上游大流量冲击，这个在 `Stream` 编程中是无法做到的，想要做到这个，就需要可以控制流速，那秘密看起来也就是在 `request(long n)` 中。

![Subscriber&Publisher&Subscription](https://s1.ax1x.com/2018/06/27/PPboXq.png)


## 他们如何工作
我们尝试使用最简单的一个例子进行我们的 `探险之旅`

```java
Flux.just("tom", "jack", "allen")
    .map(s-> s.concat("@qq.com"))
    .subscribe(System.out::println);
```
我们仅仅是将 `String` 对象进行增加一个邮箱后缀，然后再打印出来，这是一个非常简单的逻辑。

### 声明阶段
```java
//reactor.core.publisher.Flux#fromArray
public static <T> Flux<T> fromArray(T[] array) {
    // 检查略
    return onAssembly(new FluxArray<>(array));
}
```
我们可以清晰的发现，`PRR` 只是将 array 包裹成了一个 `FluxArray` 对象，我们来看看它的声明。

```java
final class FluxArray<T> extends Flux<T> implements Fuseable, Scannable {
	final T[] array;
	@SafeVarargs
	public FluxArray(T... array) {
		this.array = Objects.requireNonNull(array, "array");
	}
}
```
在具体的实例中，`FluxArray` 也仅仅是将 `array` 储存了起来，然后就返回回来了，那我们紧接着去看看 `.map(s-> s.concat("@qq.com"))` 又做了什么。

```java
public final <V> Flux<V> map(Function<? super T, ? extends V> mapper) {
    if (this instanceof Fuseable) {
        return onAssembly(new FluxMapFuseable<>(this, mapper)); ➊
    }
    return onAssembly(new FluxMap<>(this, mapper)); ➋
}
```
在 ➊ ➋ 处，我们发现都是简单的将这个 `Function<T,V>` 包装成一个新的  `FluxMapFuseable/FluxMap` 对象返回，但是我们可以看到在 `FluxMap` 的构造函数中需要2个值

```java
FluxMap(Flux<? extends T> source, Function<? super T, ? extends R> mapper) {
    super(source);
    this.mapper = Objects.requireNonNull(mapper, "mapper");
}
```
想到了什么？这里和设计模式中的 `代理模式` 极为接近，我们每次将一个 `操作` 和 `源Publisher` 组合变成一个 `新Publisher`，到这里我们已经明白了在 `subscribe()` 之前，我们什么都没做，只是在不断的包裹 `Publisher` 将作为原始的 `Publisher` 一层又一层的返回回来。终于到了我们最为激动人心的 `subscribe()` 函数了。

### subscribe 阶段
通过一顿 `Jump Definition` 大法，我们找到
```java
//reactor.core.publisher.Flux#subscribe(reactor.core.CoreSubscriber<? super T>)
public abstract void subscribe(CoreSubscriber<? super T> actual);
```
在 Flux 的 `抽象类` 中，这是一个抽象函数，也就是函数是需要子类中实现的，那我们在上面的分析过程中，我们知道每一次 `Operator` 都会包裹出一个新的 `Flux`，那我们去找到最后一次生成的 `FluxMapFuseable` 去看看它的实现。

```java
@Override
@SuppressWarnings("unchecked")
public void subscribe(CoreSubscriber<? super R> actual) {
    if (actual instanceof ConditionalSubscriber) {
        ConditionalSubscriber<? super R> cs = (ConditionalSubscriber<? super R>) actual;
        source.subscribe(new MapFuseableConditionalSubscriber<>(cs, mapper));
        return;
    }
    source.subscribe(new MapFuseableSubscriber<>(actual, mapper));
}
```
➊ 我们暂时不去关心处理 `Fuseable` 这个对象
➋ 我们自己的 `Subscriber` 在这里被包裹成一个 `MapFuseableSubscriber` 对象，又订阅 `source`，还记得 `source` 这个对象吗？我们当前所在的 `this` 对象是 `FluxMapFuseable`，而他的上一次的源头也就是我们的`FluxArray` 对象，这行我们就发现了 `MapFuseableSubscriber` 只是一个中间人，将我们的源头 `FluxArray` 和 我们自定义的 `Subscriber` 关联起来，通过将 `Subscriber` 包装成新的 `MapFuseableSubscriber` 的方式

那我们继续看看 `FluxArray` 是如何处理 `subscribe()` 函数的。

```java
@SuppressWarnings("unchecked")
public static <T> void subscribe(CoreSubscriber<? super T> s, T[] array) {
    if (array.length == 0) {
        Operators.complete(s);
        return;
    }
    if (s instanceof ConditionalSubscriber) {
        s.onSubscribe(new ArrayConditionalSubscription<>((ConditionalSubscriber<? super T>) s, array)); ➊
    }
    else {
        s.onSubscribe(new ArraySubscription<>(s, array)); ➋
    }
}
```
熟悉的味道，➊ ➋ 将 `Subscriber` 和 `Publisher` 包裹成一个 `Subscription` 对象，并将其 作为`onSubscribe` 函数调用的对象，这样的话，我们就可以完整的理解，为什么 [`Nothing Happens Until You subscribe()`](http://projectreactor.io/docs/core/release/reference/#reactive.subscribe) 因为实际上在我们调用 `subscribe()` 所有的方法都只是在申明对象。只有在 `subscribe` 之后才能出发 `onSubscribe` 调用。

那问题又来了 `onSubscribe` 又做了什么？那我们知道现在的这个 `s` 也就是 `MapFuseableSubscriber` 我们去看看它的 `onSubscribe` 实现就明白了。

### onSubscribe 阶段
```java
public void onSubscribe(Subscription s) {
    if (Operators.validate(this.s, s)) {
        this.s = (QueueSubscription<T>) s;
        actual.onSubscribe(this);
    }
}
```
很简单，我们又获得了我们自己所定义的 `Subscriber` 并调用它的 `onSubscribe` 函数，因为我们采用 `Lambda` 的方式生成的 `Subscriber` 所以也就是 `LambdaSubscriber` 对象，在他的实现中是如此写到

### request 阶段
```java
public final void onSubscribe(Subscription s) {
    if (Operators.validate(subscription, s)) {
        this.subscription = s;
        if (subscriptionConsumer != null) {
            try {
                subscriptionConsumer.accept(s); ➊
            }
            catch (Throwable t) {
                Exceptions.throwIfFatal(t);
                s.cancel();
                onError(t);
            }
        }
        else {
            s.request(Long.MAX_VALUE); ➋
        }
    }
}

```
无论是 ➊ 还是 ➋ 最为核心的都是调用了 `Subscription.request()` 函数，还记这个 `Subscription` 吗？也就是我们上一步的 `MapFuseableSubscriber`
```java
@Override
public void request(long n) {
    s.request(n);
}
```
这这里的S又是我们最外围的 `FluxArray`，我们继续查看下去。

```java
public void request(long n) {
    if (Operators.validate(n)) {
        if (Operators.addCap(REQUESTED, this, n) == 0) {
            if (n == Long.MAX_VALUE) {
                fastPath(); ➊
            }
            else {
                slowPath(n); ➋
            }
        }
    }
}
```
这里进行了一个简单的优化，我们直接去阅读 `fastPath()` 函数。

### 调用阶段
```java
void fastPath() {
    final T[] a = array;
    final int len = a.length;
    final Subscriber<? super T> s = actual;

    for (int i = index; i != len; i++) { ➊
        if (cancelled) { return; }
        T t = a[i];
        if (t == null) { /** skip **/}
        s.onNext(t); ➋
    }

    /** skip **/
    s.onComplete();
}

```
这个函数非常的简单，核心也就是一个循环体 ➊，我们在 ➋ 看出我们最终处理单一元素的 `onNext()` 函数，而这个 s 对象是 `FluxMapFuseable` 对象，在它的 onNext() 中

```java
@Override
public void onNext(T t) {
    R v;
    try {
        v = Objects.requireNonNull(mapper.apply(t), "The mapper returned a null value."); ➊
    }
    catch (Throwable e) {//skip
    }

    actual.onNext(v); ➋ 
}
```
在 ➊ 处进行 Mapper 变形
在 ➋ 将 变形之后的结构传递给下一个 `Subscriber`
这里的 `actual` 也就是我们的自己所写的 `Subscriber`

### 小结一下
1. 声明阶段: 当我们每进行一次 `Operator` 操作 （也就 map filter flatmap），就会将原有的 `FluxPublisher` 包裹成一个新的 `FluxPublisher`
![transfer](https://s1.ax1x.com/2018/06/29/PiLJN8.png)
最后生成的对象是这样的
![rs](https://s1.ax1x.com/2018/06/29/PiLCc9.png)

2. subscribe阶段: 当我们最终进行 `subscribe` 操作的时候，就会从最外层的 `Publisher` 一层一层的处理，从这层将 `Subscriber` 变化成需要的 `Subscriber` 直到最外层的 `Publisher`
![transfer](https://s1.ax1x.com/2018/06/29/PiLN9g.png)
最后生成的对象是这样的
![rs](https://s1.ax1x.com/2018/06/29/PiOBxH.png)

3. onSubscribe阶段: 在最外层的 `Publisher` 的时候调用 上一层 `Subscriber` 的 `onSubscribe` 函数，在此处将 `Publisher` 和 `Subscriber` 包裹成一个 `Subscription` 对象作为 `onSubscribe` 的入参数。
![last](https://s1.ax1x.com/2018/06/29/PivrM4.md.png)

4. 最终在 原始 `Subscriber` 对象调用 `request()` ，触发 `Subscription` 的 `Source` 获得数据作为 `onNext` 的参数，但是注意 `Subscription` 包裹的是我们封装的 `Subscriber` 所有的数据是从 `MapSubscriber` 进行一次转换再给我们的原始 `Subscriber` 的。
![Work](https://s1.ax1x.com/2018/06/29/PixEWT.md.png)

经过一顿分析，整个 `PRR` 是如何将操作整合起来的，我们已经有一个大致的了解，通过不断的包裹出新的 `Subscriber` 对象，在最终的 `request()` 行为中触发整个消息的处理，这个过程非常像 `俄罗斯套娃`，一层一层的将变化组合形变操作变成一个新的 `Subscriber`， 然后就和一个管道一样，一层一层的往下传递。

5. 最终在 `Subscription` 开始了我们整个系统的数据处理
![onNext](https://s1.ax1x.com/2018/06/29/PixelF.png)

### 其他
读者在自行阅读代码的时候可以使用 `Mono` 进行分析，也会比较简单些，比如使用下面的代码：

```java
Mono.just("tom")
    .map(s -> s.concat("123"))
    .filter(s -> s.length() > 5)
    .subscribe(System.out::println);
```

---

## 线程切换 Schedulers
整个 `PPR` 和 竞品 `RxJava` 除了变化操作以外，最为重要的就是 `线程切换`, `PPR`内置了4中线程调度器。

- 使用当前线程的 `Schedulers.immediate()`
- 一个被复用的单一线程 `Schedulers.single()`，如果希望是创建新线程的模式请使用 `Schedulers.newSingle()`
- 一个弹性线程池 `Schedulers.elastic()`，这个非常适合处理一些 ` I/O blocking` 实践
- 一个固定Wokers的线程池 `Schedulers.parallel()` 这个会创建和 `CPU` 核心数一样的线程池

`PPR` 进行线程切换的函数也很简单，只有两个
- `publishOn` : 这个函数将影响之后的操作
- `subscribeOn` : 这个函数仅仅影响 `事件源` 所在的线程

### 举个例子
```java
Flux.just("tom")
    .map(s -> {
        System.out.println("(concat @qq.com) at [" + Thread.currentThread() + "]");
        return s.concat("@qq.com");
    })
    .publishOn(Schedulers.newSingle("thread-a"))
    .map(s -> {
        System.out.println("(concat foo) at [" + Thread.currentThread() + "]");
        return s.concat("foo");
    })
    .filter(s -> {
        System.out.println("(startsWith f) at [" + Thread.currentThread() + "]");
        return s.startsWith("t");
    })
    .publishOn(Schedulers.newSingle("thread-b"))
    .map(s -> {
        System.out.println("(to length) at [" + Thread.currentThread() + "]");
        return s.length();
    })
    .subscribeOn(Schedulers.newSingle("source"))
    .subscribe(System.out::println);
```
执行的结果如下

```text
(concat @qq.com) at [Thread[source-1,5,main]]  ➊
(concat foo) at [Thread[thread-a-3,5,main]] ➋
(startsWith f) at [Thread[thread-a-3,5,main]] ➌
(to length) at [Thread[thread-b-2,5,main]] ➍
```
➊ 正对应着  `.subscribeOn(Schedulers.newSingle("source"))` 在一个 `source` 线程开始事件，和顺序无关
➋ ➌ 都对应着 `.publishOn(Schedulers.newSingle("thread-a"))` 在一个 `thread-a` 进行操作
➍ 不言而喻

### 工作原理
其中会有什么黑科技呢，在阅读源码之前，我也有这样的思考。我们一顿 `Jump` 我们又看见了一个熟悉的身影。
```java
final Flux<T> publishOn(Scheduler scheduler, boolean delayError, int prefetch, int lowTide) {
    if (this instanceof Callable) {
        if (this instanceof Fuseable.ScalarCallable) {
            @SuppressWarnings("unchecked")
            Fuseable.ScalarCallable<T> s = (Fuseable.ScalarCallable<T>) this;
            try {
                return onAssembly(new FluxSubscribeOnValue<>(s.call(), scheduler)); ➊
            }
            catch (Exception e) {
                //leave FluxSubscribeOnCallable defer exception call
            }
        }
        @SuppressWarnings("unchecked")
        Callable<T> c = (Callable<T>)this;
        return onAssembly(new FluxSubscribeOnCallable<>(c, scheduler)); ➋
    }

    return onAssembly(new FluxPublishOn<>(this, scheduler, delayError, prefetch, lowTide, Queues.get(prefetch))); ➌
}
```
暂且不管其他的部分，我们发现核心的依然是 构建了一个新的包裹 `Subscribe`，我们去看看最为简单的 `FluxPublishOn`，这里仅仅是 `New` 出一个新的对象，至于正在的运行时，在这里是无从得知的。但是我们去看看它的定义发现一个有钱的函数

```java
@Override
@SuppressWarnings("unchecked")
public void subscribe(CoreSubscriber<? super T> actual) {
    Worker worker = Objects.requireNonNull(scheduler.createWorker(),
					"The scheduler returned a null worker");
    //阉割
    if (actual instanceof ConditionalSubscriber) {
        ConditionalSubscriber<? super T> cs = (ConditionalSubscriber<? super T>) actual;
        source.subscribe(new PublishOnConditionalSubscriber<>(cs,
                scheduler,
                worker,
                delayError,
                prefetch,
                lowTide,
                queueSupplier));
        return;
    }
    source.subscribe(new PublishOnSubscriber<>(actual,
            scheduler,
            worker,
            delayError,
            prefetch,
            lowTide,
            queueSupplier));
}
```
我们发现在 `subscribe()` 函数中，我们在构建一个新的 `PublishOnSubscriber` 对象时候，我们将 `worker` 传入其中，我们又将 `actual` 上一层的 `Subscriber` 传入，在这里可以推测出，我们又将 `Subscriber` 包装了。

而我们已经知道了事件的触发处是 `request()`，和传递元素处是 `onNext()` 那线程切换自然也在这两处地方。我们找到源码。
```java
//reactor.core.publisher.FluxPublishOn.PublishOnSubscriber#request
@Override
public void request(long n) {
    if (Operators.validate(n)) {
        Operators.addCap(REQUESTED, this, n);
        //WIP also guards during request and onError is possible
        trySchedule(this, null, null); ➊
    }
}

//reactor.core.publisher.FluxPublishOn.PublishOnSubscriber#onNext
public void onNext(T t) { 
    if (!queue.offer(t)) {  ➋ 
            error = Operators.onOperatorError(s, Exceptions.failWithOverflow(Exceptions.BACKPRESSURE_ERROR_QUEUE_FULL), t,
                    actual.currentContext());
            done = true;
    }
    trySchedule(this, null, t); ➌
}
```
➊➌ 处出现的 trySchedule() 也就是切换线程的核心。
➋ 处有个值得注意的地方，我们将当前的接受的到的元素放置到了一个队列之中

```java
void trySchedule(@Nullable Subscription subscription,
				@Nullable Throwable suppressed,
				@Nullable Object dataSignal) {
    try {
        worker.schedule(this);
    }
    catch (RejectedExecutionException ree) {
       //略
    }
}
```
切换的代码并不复杂，在当前的对象放到另外一个 `Worker` 中运行，而我们知道，从另外一个线程运行代码是在 `Runnable` 的 `run` 函数中
```java
//reactor.core.publisher.FluxPublishOn.PublishOnSubscriber#run
@Override
public void run() {
    if (outputFused) {
        runBackfused();
    }
    else if (sourceMode == Fuseable.SYNC) {
        runSync();
    }
    else {
        runAsync();
    }
}
```
继续看查看
```java
void runSync() {
    final ConditionalSubscriber<? super T> a = actual;
    final Queue<T> q = queue;
    long e = produced;
    for (;;) {
        long r = requested;
        while (e != r) {
            T v;
            try { v = q.poll(); ➊ }
            catch (Throwable ex) { /* skip */}

            if (v == null) { doComplete(a); return; }

            if (a.tryOnNext(v)) { ➋
                e++;
            }
        }

        if (q.isEmpty()) {
            doComplete(a);
            return;
        }
    }
}
```
➊ 在切换线程之前，我们将我们需要处理的元素存储在我们的 `Queue`队列之中，在这里取出来
➋ 将我们的数据交付给真实操作的 `Subscriber` 

### 短小结
看到这里的时候，我们已经知道了 `publishOn` 的工作原理，和 `Operator` 是类型的，只是在 `OnNext()` 的函数嵌入线程处理的代码，为什么 `publishOn` 仅对后续的操作有效，我们也可以看出来，因为 `publishOn` 的原理如下图。

<div style="width:100%; overflow-y:scroll;" id="diagram"></div>
<script>
	var data =
	['Title: publishOn原理时序图',
	 'Source->FluxPublishOn: 1. onNext 将输入元素置于 Queue',
	 'FluxPublishOn->FluxPublishOn: 2.trySchedule 切换线程',
	 'FluxPublishOn->FluxPublishOn: 3. run 取出 Queue 继续处理',
     'FluxPublishOn->Subscriber: 4. 后续Subscriber处理'
    ].join('\n');
  	var diagram = Diagram.parse(data);
  	diagram.drawSVG("diagram", {theme: 'simple', scale: 0.5});
</script>

用图表示的话是这样的：
![pic](https://s1.ax1x.com/2018/06/29/PFp8f0.png)


所以我们知道了，`publishOn` 并不会关心之前的操作是在哪里进行的，到它这里的时候，会切换一次线程即可。

### subscribeOn 原理
因为从上文的描述中，我们得知了 `subscribeOn` 修改的是 `Source` 的行为，在几个关键的函数中，大概率就是只有 `subscribe()` 阶段可以修改这个，果不其然。
```java
@Override
@SuppressWarnings("unchecked")
public void subscribe(CoreSubscriber<? super T> actual) {
    Worker worker = Objects.requireNonNull(scheduler.createWorker(), "The scheduler returned a null Function");
    SubscribeOnSubscriber<T> parent = new SubscribeOnSubscriber<>(source,
            actual, worker, requestOnSeparateThread);
    
    actual.onSubscribe(parent); ➊
    try { worker.schedule(parent); ➋ }
    
    catch (RejectedExecutionException ree) { /* skip */ }
}
```
在 ➊ 处，构建了一个新的 `Subscriber` 叫 `SubscribeOn`，将 `actual` 包裹成一个新的对象 ，然后在 ➋ 处进行了 线程的切换。而它的 `Run` 也很简单

```java
@Override
public void run() {
    THREAD.lazySet(this, Thread.currentThread());
    source.subscribe(this);
}
```
我们看出来，`subscribeOn` 相对来说更为简单，我们 创建一个新的 `Subscriber` 将 `源Subscriber` 关联起来，然后切换线程，之后就是正常的运行机制。

<div style="width:100%; overflow-y:scroll;" id="diagram2"></div>
<script>
	var data2 =
	['Title: subscribeOn原理时序图',
	 'Source->FluxSubscribeOn: 1. subscribe 构建一个 SubscribeOnSubscriber，关联源 Subscriber',
	 'FluxSubscribeOn->FluxSubscribeOn: 2.schedule 切换线程',
	 'FluxSubscribeOn->Source: 3. subscribe 继续远逻辑处理',
    ].join('\n');
  	var diagram2 = Diagram.parse(data2);
  	diagram2.drawSVG("diagram2", {theme: 'simple', scale: 0.5});
</script>

用图表示的话是这样的：
![pic](https://s1.ax1x.com/2018/06/29/PFCrQK.png)

### 谜题

如果我们写出形如下面的代码，那结果应该如何。
```java
Flux.just("tom")
    .map(s -> {
        System.out.println("(to length) at [" + Thread.currentThread() + "]");
        return s.length();
    })
    .subscribeOn(Schedulers.newSingle("source1"))
    .subscribeOn(Schedulers.newSingle("source2"))
    .subscribeOn(Schedulers.newSingle("source3"))
    .subscribeOn(Schedulers.newSingle("source4"))
    .subscribe(System.out::println);
```
答案是:
```bash
(to length) at [Thread[source1-4,5,main]]
```
我们发现下面的 `.subscribeOn()` 根本无效，让我们会议一下，`Subscriber` 的包装是 逆序的。 包装的顺序是 `源Subscriber` `Subscriber-Source-1` `Subscriber-Source-2` `Subscriber-Source-3` `Subscriber-Source-4` 那我们在 `Subscriber-Source-4` 阶段进行了线程切换，之后到了 `Subscriber-Source-3` 又进行了一次切换 所以只有最后一次 `Subscriber-Source-1` 最终影响我们的 `Source` 事件所在的线程。

![pic](https://s1.ax1x.com/2018/06/29/PFChWt.png)

## 大总结
我们最后总结一下

1. 在声明阶段，我们像 `俄罗斯套娃` 一样，创建一个嵌套一个的 `Publisher`
2. 在 `subscribe()` 调用的时候，我们从最外围的  `Publisher` 根据包裹的 `Operator` 创建各种 `Subscriber`
3. `Subscriber` 通过 `onSubscribe()` 使得他们像一个链条一样关联起来，并和 最外围的 `Publisher` 组合成一个 `Subscription`
4. 在最底层的 `Subscriber` 调用 `onSubscribe` 去调用 `Subscription` 的 `request(n);` 函数开始操作
5. 元素就像水管中的水一样挨个 经过  `Subscriber` 的 `onNext()`，直至我们最终消费的 `Subscriber`
 

## 参考文档
- [Reactor 3 Reference](http://projectreactor.io/docs/core/release/reference/#intro-reactive)
- [Reactor 3 参考文档](https://htmlpreview.github.io/)
- [Reactor 指南中文版](http://projectreactor.mydoc.io/?t=44474)
- [Reactive Programming with JDK 9 Flow API](https://community.oracle.com/docs/DOC-1006738)
- [Reactive Stream 各实现的对比（一）](https://blog.piasy.com/AdvancedRxJava/2017/03/19/comparison-of-reactive-streams-part-1/)
- [observer-design-pattern-in-java](http://www.codenuclear.com/observer-design-pattern-in-java/)


## 附录
![BigPic](https://s1.ax1x.com/2018/06/27/PP7x54.png)