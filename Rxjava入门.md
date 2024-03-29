# Rxjava学习总结
------
参考：
[给 Android 开发者的 RxJava 详解-扔物线](https://gank.io/post/560e15be2dca930e00da1083#toc_19)
[深入浅出RxJava](http://blog.csdn.net/lzyzsd/article/details/41833541)

##基础
"a library for composing asynchronous and event-based programs using observable sequences for the Java VM" （一个在 Java VM 上使用可观测的序列来组成==异步==的、基于事件的程序的库）

采用函数响应式编程的思想，使异步代码更简洁、逻辑更清晰，避免了令人头痛的回调的嵌套、线程调度等问题。

###RxJava 的观察者模式
RxJava 有四个基本概念：Observable (被观察者)、 Subscriber (观察者)、 subscribe (订阅)、事件。

Observable 和 Subscriber 通过 subscribe() 方法实现订阅关系，从而 Observable 可以在需要的时候发出事件来通知 Subscriber。

与传统观察者模式的区别是：

- 如果 Observerble 没有任何的 Subscriber，那么这个 Observable 不会发出任何事件的。
- RxJava 的事件回调方法除了普通事件 onNext() （相当于 onClick() / onEvent()）之外，还定义了两个特殊的事件：onCompleted() 和 onError()。
![](/Users/dengfa/MyZone/macDown/img/52eb2279jw1f2rx46dspqj20gn04qaad.jpg)
- onCompleted(): 事件队列完结。当事件流发送完毕时，会触发 onCompleted() 方法。
- onError(): 事件队列异常。在事件处理过程中有异常抛出时，onError() 会被触发，同时队列自动终止，不允许再有事件发出。(RxJava 不仅把每个事件单独处理，还会把它们看做一个队列。)

##基本实现
###1.创建Observable
~~~java
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello");
        subscriber.onNext("Hi");
        subscriber.onNext("Aloha");
        subscriber.onCompleted();
    }
});
~~~
Observable 即被观察者，它决定什么时候触发事件以及触发怎样的事件。
这里，使用`create()`方法来创建一个 Observable，并定义事件触发规则：观察者Subscriber 将会被调用三次 `onNext()` 和一次 `onCompleted()`。这里传入了一个 OnSubscribe 对象作为参数。OnSubscribe 会被存储在返回的 Observable 对象中，它的作用相当于一个计划表，当 Observable 被订阅的时候，OnSubscribe 的 `call()` 方法会自动被调用，事件序列就会依照设定依次触发。这样，由被观察者调用了观察者的回调方法，就实现了由被观察者向观察者的事件传递，即观察者模式。

`create()`方法是 RxJava 最基本的创造事件序列的方法。基于这个方法， RxJava 还提供了一些方法用来快捷创建事件队列，例如：

- `just(T...)`: 将传入的参数依次发送出来。
- `from(T[]) / from(Iterable<? extends T>)` : 将传入的数组或 Iterable 拆分成具体对象后，依次发送出来。

~~~java
Observable observable = Observable.just("Hello", "Hi", "Aloha");
~~~
~~~java
String[] words = {"Hello", "Hi", "Aloha"};
Observable observable = Observable.from(words);
~~~
这两段代码与前面的创建方式等价。

###2.创建Observer
Observer 即观察者，它决定事件触发的时候将有怎样的行为。 
RxJava 中的 Observer 接口： 

~~~java
public interface Observer<T> {

    void onCompleted();

    void onError(Throwable e);

    void onNext(T t);

}
~~~
此外，RxJava 还内置了一个实现了 Observer、Subscription 的抽象类：Subscriber。 
Subscriber 对 Observer 接口进行了一些扩展： 
1. `onStart()`: 在 subscribe 刚开始，而事件还未发送之前被调用，可以用于做一些准备工作，例如数据的清零或重置。需要注意的是，如果对准备工作的线程有要求（例如弹出一个显示进度的对话框，这必须在主线程执行），`onStart()`就不适用了，因为它总是在 subscribe 所发生的线程被调用，而不能指定线程。要在指定的线程来做准备工作，可以使用`doOnSubscribe()`方法。
2. `unsubscribe()`: 这是 Subscriber 所实现的另一个接口 Subscription 的方法，用于取消订阅。在这个方法被调用后，Subscriber 将不再接收事件。一般在这个方法调用前，可以使用`isUnsubscribed()`先判断一下状态。`unsubscribe()`这个方法很重要，因为在 `subscribe()`之后， Observable 会持有 Subscriber 的引用，这个引用如果不能及时被释放，将有内存泄露的风险。所以最好保持一个原则：要在不再使用的时候尽快在合适的地方（例如`onPause()``onStop()`等方法中）调用 unsubscribe() 来解除引用关系，以避免内存泄露的发生。

###3.Subscribe
通过subscribe()方法将被观察者和观察者关联起来： 

~~~java
observable.subscribe(subscriber);
~~~
内部实现（仅核心代码）： 

~~~java
public Subscription subscribe(Subscriber subscriber) {
    subscriber.onStart();
    onSubscribe.call(subscriber);
    return subscriber;
}
~~~
`subscriber()`主要做了3件事： 
1. 调用`Subscriber.onStart()`，进行订阅前的初始化工作。
2. 调用 Observable 中的 `OnSubscribe.call(Subscriber)`。事件发送的逻辑开始运行。可以看到， Observable 并不是在创建的时候就立即开始发送事件，而是在它被订阅的时候，即当`subscribe()`方法执行的时候，才开始发送事件。
3. 将传入的 Subscriber 作为 Subscription 返回，便于解除订阅`unsubscribe()`.
![](/Users/dengfa/MyZone/macDown/img/52eb2279jw1f2rx4ay0hrg20ig08wk4q.gif)

##线程控制

默认情况下，事件的发出和 消费都是在同一个线程的。 
要实现『后台处理，前台回调』的异步机制，可以使用`subscribeOn()`和`observeOn()`两个方法，结合Scheduler，来对线程进行控制。 

- `subscribeOn()`: 指定`subscribe()`所发生的线程，即`Observable.OnSubscribe`被激活时所处的线程。或者叫做事件产生的线程。 
- `observeOn()`: 指定 Subscriber 所运行的线程。或者叫做事件消费的线程。

RxJava 内置的 Scheduler： 

- `Schedulers.immediate()`: 直接在当前线程运行，这是默认的 Scheduler。
- `Schedulers.newThread()`: 总是启用新线程，并在新线程执行操作。
- `Schedulers.io()`: I/O 操作（读写文件、读写数据库、网络信息交互等）。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是使用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。注意不要把计算工作放在 io() 线程中，以避免创建不必要的线程。
- `Schedulers.computation()`: 用于 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用固定的线程池，大小为 CPU 核数。注意不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。
- `AndroidSchedulers.mainThread()`：Android 主线程。

示例（适用于多数的 『后台线程取数据，主线程显示』的程序策略）：

~~~java
Observable.just(1, 2, 3, 4)
    .subscribeOn(Schedulers.io()) // 指定 subscribe() 发生在 IO 线程
    .observeOn(AndroidSchedulers.mainThread()) // 指定 Subscriber 的回调发生在主线程
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer number) {
            Log.d(tag, "number:" + number);
        }
    });
~~~

**线程的自用控制：**
observeOn() 指定的是 Subscriber 的线程，而这个 Subscriber 并不一定是subscribe() 参数中的 Subscriber ，而是 observeOn() 执行时的当前 Observable 所对应的Subscriber ，即它的直接下级 Subscriber 。换句话说，observeOn() 指定的是它之后的操作所在的线程。因此如果有多次切换线程的需求，只要在每个想要切换线程的位置调用一次 observeOn() 即可。

~~~java
Observable.just(1, 2, 3, 4) // IO 线程，由 subscribeOn() 指定
    .subscribeOn(Schedulers.io())
    .observeOn(Schedulers.newThread())
    .map(mapOperator) // 新线程，由 observeOn() 指定
    .observeOn(Schedulers.io())
    .map(mapOperator2) // IO 线程，由 observeOn() 指定
    .observeOn(AndroidSchedulers.mainThread) 
    .subscribe(subscriber);  // Android 主线程，由 observeOn() 指定
~~~
subscribeOn() 和 observeOn() 的内部实现，也是基于 lift() 方法。
subscribeOn() 原理图：
![](/Users/dengfa/MyZone/我的博客/img/52eb2279jw1f2rxcynbsuj20ha0d7wg2.jpg)
observeOn() 原理图：
![](/Users/dengfa/MyZone/我的博客/img/52eb2279jw1f2rxd05lttj20hj0cyabl.jpg)

从图中可以看出，subscribeOn() 和 observeOn() 都做了线程切换的工作（图中的 "schedule..." 部位）。不同的是， subscribeOn() 的线程切换发生在 OnSubscribe 中，即在它通知上一级 OnSubscribe 时，这时事件还没有开始发送，因此 subscribeOn() 的线程控制可以从事件发出的开端就造成影响；而 observeOn() 的线程切换则发生在它内建的 Subscriber 中，即发生在它即将给下一级 Subscriber 发送事件时，因此 observeOn() 控制的是它后面的线程。

多个 subscribeOn() 和 observeOn() 混合使用时，线程调度的流程图：
![](/Users/dengfa/MyZone/我的博客/img/52eb2279jw1f2rxd1vl7xj20hd0hzq6e.jpg)
图中共有 5 处含有对事件的操作。由图中可以看出，①和②两处受第一个 subscribeOn() 影响，运行在红色线程；③和④处受第一个 observeOn() 的影响，运行在绿色线程；⑤处受第二个 onserveOn() 影响，运行在紫色线程；而第二个 subscribeOn() ，由于在通知过程中线程就被第一个 subscribeOn() 截断，因此对整个流程并没有任何影响。这里也就回答了前面的问题：==当使用了多个 subscribeOn() 的时候，只有第一个 subscribeOn() 起作用。==

##事件变换
RxJava 提供了对事件序列进行变换的支持，这是它的核心功能之一，将事件序列中的对象或整个序列进行加工处理，转换成不同的事件或事件序列。
###lift:针对事件项和事件序列的变换
`map()`、`flatMap()`等变换都是基于同一个基础的变换方法`lift(Operator)`的包装。 
 
`lift()`的内部实现（仅核心代码）： 

~~~java
public <R> Observable<R> lift(Operator<? extends R, ? super T> operator) {
    return Observable.create(new OnSubscribe<R>() {
        @Override
        public void call(Subscriber subscriber) {
            Subscriber newSubscriber = operator.call(subscriber);
            newSubscriber.onStart();
            onSubscribe.call(newSubscriber);
        }
    });
}
~~~
1.lift() 创建了一个 Observable 后，加上之前的原始 Observable，已经有两个 Observable 了； 
2.而同样地，新 Observable 里的新 OnSubscribe 加上之前的原始 Observable 中的原始 OnSubscribe，也就有了两个 OnSubscribe； 
3.当用户调用经过 lift() 后的 Observable 的 subscribe() 的时候，使用的是 lift() 所返回的新的 Observable ，于是它所触发的 onSubscribe.call(subscriber)，也是用的新 Observable 中的新 OnSubscribe，即在 lift() 中生成的那个 OnSubscribe； 
4.而这个新 OnSubscribe 的 call() 方法中的 onSubscribe ，就是指的原始 Observable 中的原始 OnSubscribe ，在这个 call() 方法里，新 OnSubscribe 利用 operator.call(subscriber) 生成了一个新的 Subscriber（Operator 就是在这里，通过自己的 call() 方法将新 Subscriber 和原始 Subscriber 进行关联，并插入自己的『变换』代码以实现变换），然后利用这个新 Subscriber 向原始 Observable 进行订阅。 
这样就实现了 lift() 过程，有点像一种代理机制，通过事件拦截和处理实现事件序列的变换。

简而言之，在 Observable 执行了 lift(Operator) 方法之后，会返回一个新的 Observable，这个新的 Observable 会像一个代理一样，负责接收原始的 Observable 发出的事件，并在处理后发送给 Subscriber。
![](/Users/dengfa/MyZone/我的博客/img/52eb2279jw1f2rxcu9f46g20go0cz4qp.gif)
两次和多次的 lift() 同理：
![](/Users/dengfa/MyZone/我的博客/img/52eb2279jw1f2rxcvophmj20h30hl0v3.jpg)

示例：将事件中的 Integer 对象转换成 String

~~~java
observable.lift(new Observable.Operator<String, Integer>() {
    @Override
    public Subscriber<? super Integer> call(final Subscriber<? super String> subscriber) {
        // 将事件序列中的 Integer 对象转换为 String 对象
        return new Subscriber<Integer>() {
            @Override
            public void onNext(Integer integer) {
                subscriber.onNext("" + integer);
            }

            @Override
            public void onCompleted() {
                subscriber.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                subscriber.onError(e);
            }
        };
    }
});
~~~

###compose: 对 Observable 整体的变换
它和 lift() 的区别在于， lift() 是针对事件项和事件序列的，而 compose() 是针对 Observable 自身进行变换。 

示例： 
假设在程序中有多个 Observable ，并且他们都需要应用一组相同的 lift() 变换。你可以这么写：

~~~java
observable1
    .lift1()
    .lift2()
    .lift3()
    .lift4()
    .subscribe(subscriber1);
observable2
    .lift1()
    .lift2()
    .lift3()
    .lift4()
    .subscribe(subscriber2);
observable3
    .lift1()
    .lift2()
    .lift3()
    .lift4()
    .subscribe(subscriber3);
observable4
    .lift1()
    .lift2()
    .lift3()
    .lift4()
    .subscribe(subscriber1);
~~~
这样代码复用性太差，于是改写成：

~~~java
private Observable liftAll(Observable observable) {
    return observable
        .lift1()
        .lift2()
        .lift3()
        .lift4();
}
...
liftAll(observable1).subscribe(subscriber1);
liftAll(observable2).subscribe(subscriber2);
liftAll(observable3).subscribe(subscriber3);
liftAll(observable4).subscribe(subscriber4);
~~~
可读性、可维护性都提高了。可是 Observable 被一个方法包起来，如果后期需求变化，observable1在进行liftAll()变换前需要先进行lift5()变换，就打破了api的链式调用，这种方式对于 Observale 的灵活性增添了限制。此时，就可以用 compose() 来解决：

~~~java
public class LiftAllTransformer implements Observable.Transformer<Integer, String> {
    @Override
    public Observable<String> call(Observable<Integer> observable) {
        return observable
            .lift1()
            .lift2()
            .lift3()
            .lift4();
    }
}
...
Transformer liftAll = new LiftAllTransformer();
observable1.compose(liftAll).subscribe(subscriber1);
observable2.compose(liftAll).subscribe(subscriber2);
observable3.compose(liftAll).subscribe(subscriber3);
observable4.compose(liftAll).subscribe(subscriber4);
~~~

##操作符
[Rx中文文档](https://mcxiaoke.gitbooks.io/rxdocs/content/Operators.html)
[操作符的使用示例](http://wiki.jikexueyuan.com/project/rxjava//chapter4/filtering_a_sequence.html)
###创建操作
- Create — 通过调用观察者的方法从头创建一个Observable
- Defer — 在观察者订阅之前不创建这个Observable，为每一个观察者创建一个新的Observable
- Empty/Never/Throw — 创建行为受限的特殊Observable
- From — 将其它的对象或数据结构转换为Observable
- Interval — 创建一个定时发射整数序列的Observable
- Just — 将对象或者对象集合转换为一个会发射这些对象的Observable
- Range — 创建发射指定范围的整数序列的Observable
- Repeat — 创建重复发射特定的数据或数据序列的Observable
- Start — 创建发射一个函数的返回值的Observable
- Timer — 创建在一个指定的延迟之后发射单个数据的Observable

###变换操作
- Buffer — 缓存，可以简单的理解为缓存，它定期从Observable收集数据到一个集合，然后把这些数据集合打包发射，而不是一次发射一个
- FlatMap — 扁平映射，将Observable发射的数据变换为Observables集合，然后将这些Observable发射的数据平坦化的放进一个单独的Observable，可以认为是一个将嵌套的数据结构展开的过程。
- GroupBy — 分组，将原来的Observable分拆为Observable集合，将原始Observable发射的数据按Key分组，每一个Observable发射一组不同的数据
- Map — 映射，通过对序列的每一项都应用一个函数变换Observable发射的数据，实质是对序列中的每一项执行一个函数，函数的参数就是这个数据项
- Scan — 扫描，对Observable发射的每一项数据应用一个函数，然后按顺序依次发射这些值
- Window — 窗口，定期将来自Observable的数据分拆成一些Observable窗口，然后发射这些窗口，而不是每次发射一项。类似于Buffer，但Buffer发射的是数据，Window发射的是Observable，每一个Observable发射原始Observable的数据的一个子集

###过滤操作
- Debounce — 只有在空闲了一段时间后才发射数据，通俗的说，就是如果一段时间没有操作，就执行一次操作
- Distinct — 去重，过滤掉重复数据项
- ElementAt — 取值，取特定位置的数据项
- Filter — 过滤，过滤掉没有通过谓词测试的数据项，只发射通过测试的
- First — 首项，只发射满足条件的第一条数据
- IgnoreElements — 忽略所有的数据，只保留终止通知(onError或onCompleted)
- Last — 末项，只发射最后一条数据
- Sample — 取样，定期发射最新的数据，等于是数据抽样，有的实现里叫ThrottleFirst
- Skip — 跳过前面的若干项数据
- SkipLast — 跳过后面的若干项数据
- Take — 只保留前面的若干项数据
- TakeLast — 只保留后面的若干项数据

###组合操作
- And/Then/When — 通过模式(And条件)和计划(Then次序)组合两个或多个Observable发射的数据集
- CombineLatest — 当两个Observables中的任何一个发射了一个数据时，通过一个指定的函数组合每个Observable发射的最新数据（一共两个数据），然后发射这个函数的结果
- Join — 无论何时，如果一个Observable发射了一个数据项，只要在另一个Observable发射的数据项定义的时间窗口内，就将两个Observable发射的数据合并发射
- Merge — 将两个Observable发射的数据组合并成一个
- StartWith — 在发射原来的Observable的数据序列之前，先发射一个指定的数据序列或数据项
- Switch — 将一个发射Observable序列的Observable转换为这样一个Observable：它逐个发射那些Observable最近发射的数据
- Zip — 打包，使用一个指定的函数将多个Observable发射的数据组合在一起，然后将这个函数的结果作为单项数据发射

###错误处理
这些操作符用于从错误通知中恢复：

- Catch — 捕获，继续序列操作，将错误替换为正常的数据，从onError通知中恢复
- Retry — 重试，如果Observable发射了一个错误通知，重新订阅它，期待它正常终止 

###辅助操作
- Delay — 延迟一段时间发射结果数据
- Do — 注册一个动作占用一些Observable的生命周期事件，相当于Mock某个操作
- Materialize/Dematerialize — 将发射的数据和通知都当做数据发射，或者反过来
- ObserveOn — 指定观察者观察Observable的调度程序（工作线程）
- Serialize — 强制Observable按次序发射数据并且功能是有效的
- Subscribe — 收到Observable发射的数据和通知后执行的操作
- SubscribeOn — 指定Observable应该在哪个调度程序上执行
- TimeInterval — 将一个Observable转换为发射两个数据之间所耗费时间的Observable
- Timeout — 添加超时机制，如果过了指定的一段时间没有发射数据，就发射一个错误通知
- Timestamp — 给Observable发射的每个数据项添加一个时间戳
- Using — 创建一个只在Observable的生命周期内存在的一次性资源

###条件和布尔操作
这些操作符可用于单个或多个数据项，也可用于Observable。

- All — 判断Observable发射的所有的数据项是否都满足某个条件
- Amb — 给定多个Observable，只让第一个发射数据的Observable发射全部数据
- Contains — 判断Observable是否会发射一个指定的数据项
- DefaultIfEmpty — 发射来自原始Observable的数据，如果原始Observable没有发射数据，就发射一个默认数据
- SequenceEqual — 判断两个Observable是否按相同的数据序列
- SkipUntil — 丢弃原始Observable发射的数据，直到第二个Observable发射了一个数据，然后发射原始Observable的剩余数据
- SkipWhile — 丢弃原始Observable发射的数据，直到一个特定的条件为假，然后发射原始Observable剩余的数据
- TakeUntil — 发射来自原始Observable的数据，直到第二个Observable发射了一个数据或一个通知
- TakeWhile — 发射原始Observable的数据，直到一个特定的条件为真，然后跳过剩余的数据

###算术和聚合操作
这些操作符可用于整个数据序列

- Average — 计算Observable发射的数据序列的平均值，然后发射这个结果
- Concat — 不交错的连接多个Observable的数据
- Count — 计算Observable发射的数据个数，然后发射这个结果
- Max — 计算并发射数据序列的最大值
- Min — 计算并发射数据序列的最小值
- Reduce — 按顺序对数据序列的每一个应用某个函数，然后返回这个值
- Sum — 计算并发射数据序列的和

###连接操作
一些有精确可控的订阅行为的特殊Observable

- Connect — 指示一个可连接的Observable开始发射数据给订阅者
- Publish — 将一个普通的Observable转换为可连接的
- RefCount — 使一个可连接的Observable表现得像一个普通的Observable
- Replay — 确保所有的观察者收到同样的数据序列，即使他们在Observable开始发射数据之后才订阅

###转换操作
- To — 将Observable转换为其它的对象或数据结构
- Blocking - 阻塞Observable的操作符


##使用场景
- 与 Retrofit 的结合
- RxBinding
- 各种异步操作
- RxBus


##其他
###subscribe()不完整定义的回调，Action、Function接口

###doOnSubscribe()
Subscriber 的 onStart() 可以用作流程开始前的初始化。然而 onStart() 由于在 subscribe() 发生时就被调用了，因此不能指定线程，而是只能执行在 subscribe() 被调用时的线程。这就导致如果 onStart() 中含有对线程有要求的代码（例如在界面上显示一个 ProgressBar，这必须在主线程执行），将会有线程非法的风险，因为有时你无法预测 subscribe() 将会在什么线程执行。

而与 Subscriber.onStart() 相对应的，有一个方法 Observable.doOnSubscribe() 。它和 Subscriber.onStart() 同样是在 subscribe() 调用后而且在事件发送前执行，但区别在于它可以指定线程。默认情况下， doOnSubscribe() 执行在 subscribe() 发生的线程；而如果在 doOnSubscribe() 之后有 subscribeOn() 的话，它将执行在离它最近的 subscribeOn() 所指定的线程。

~~~java
Observable.create(onSubscribe)
    .subscribeOn(Schedulers.io())
    .doOnSubscribe(new Action0() {
        @Override
        public void call() {
            progressBar.setVisibility(View.VISIBLE); // 需要在主线程执行
        }
    })
    .subscribeOn(AndroidSchedulers.mainThread()) // 指定主线程
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(subscriber);
~~~
