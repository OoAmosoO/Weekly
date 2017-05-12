

```java
public final static <T> Observable<T> create(OnSubscribe<T> f) {
  return new Observable<T>(hook.onCreate(f));
}

protected Observable(OnSubscribe<T> f) {
  this.onSubscribe = f;
}

```

创建完了，然后就是通过subscribe(Subscriber<? super T> subscriber)进行订阅

```java
 static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
  ....
  subscriber.onStart();
  // hook.onSubscribeStart 就是返回的创建时传入的 onSubscribe， 直接调用了call
  hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
  return hook.onSubscribeReturn(subscriber);
  ....
  }
```

订阅的时候，调用了我们创建时传入的onSubscribe的call方法，在看下一般onSubscribe 怎么实现。

最简单的就是调用onnext oncomplete，如下：

```java
Observable.OnSubscribe<String> onSubscriber1 = new Observable.OnSubscribe<String>() {
  @Override
  public void call(Subscriber<? super String> subscriber) {
  subscriber.onNext("test");
  subscriber.onCompleted();
  }
};
```

还有如果支持背压的话，也有如下代码：
```java
@Override
  public void call(Subscriber<? super T> child) {
  child.setProducer(new FromArrayProducer<T>(child, array));
  }
```
默认的话，最后也是和上面一样，调用到 onNext onCompleted

上面就是一般调用的最简单的流程，但RxJava最吸引人的就是各种转换的api，这些转化就用到了lift函数

说到转化，最常用的应该就是map了

```java
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
  return lift(new OperatorMap<T, R>(func));
}

```
这里可以看到OperatorMap继承自Operator, 而Operator又继承自Func1接口，也就是说Operator接口的call方法会接收一个Subscriber类型的参数，
并且返回另外一个Subscriber类型的对象。Operator.call方法返回一个Subscriber对象，其实我们可以这么理解，每一个operator也是一个订阅者，
它返回的Subscriber对象正好用来订阅Observable发出来的消息。

有一点需要注意的是OperatorMap和Operator的泛型参数顺序刚好是相反的，为什么要这么做呢？其实很简单，因为Operator本身是对Observable发出的数据
进行转换的，所以经常会出现operator转换之后返回的数据类型变了，而OperatorMap这里刚好颠倒了一下顺序，就可以保证call方法返回的Subscriber类型
可以订阅Observable发出的数据。

然后看lift做了什么

```java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
  return new Observable<R>(new OnSubscribeLift<T, R>(onSubscribe, operator));
  }


```

lift就是new了一个Observable， 传了了一个OnSubscribeLift， 这就和最初讲的一般流程类似了，说明lift之后，唯一变化的就是onSubscribe的call方法，其中创建OnSubscribeLift时传入的参数onSubscribe就是map操作前的onSubscribe。

```java
 @Override
  public void call(Subscriber<? super R> o) {
  try {

  // operator 就是调用map时传入的转化方法Func1的一个封装,R是我们要转化的目标类型，T是转化前的类型
  // 当转化完成，订阅时 ，实际是通过operator“反向”把目标R转化生成一个转化前observal可以处理的T类型Subscriber
  // 后面的操作就跟最初的订阅流程一样了
  Subscriber<? super T> st = hook.onLift(operator).call(o);
  try {
  // new Subscriber created and being subscribed with so 'onStart' it
  st.onStart();
  parent.call(st);
  } catch (Throwable e) {
  // localized capture of errors rather than it skipping all operators
  // and ending up in the try/catch of the subscribe method which then
  // prevents onErrorResumeNext and other similar approaches to error handling
  Exceptions.throwIfFatal(e);
  st.onError(e);
  }
  } catch (Throwable e) {
  Exceptions.throwIfFatal(e);
  // if the lift function failed all we can do is pass the error to the final Subscriber
  // as we don't have the operator available to us
  o.onError(e);
  }
  }

```

可以看出call里全部的处理都在operator里怎么反向转化了,传入的Func1是 T -> R ,operator 需要实现 R -> T
```java
public final class OperatorMap<T, R> implements Operator<R, T>
```

实现都在其call方法里。

```java
@Override
  public Subscriber<? super T> call(final Subscriber<? super R> o) {
  // o是目标Subscriber ，transformer就是最初调用map时传的转化方法func1
  MapSubscriber<T, R> parent = new MapSubscriber<T, R>(o, transformer);
  o.add(parent);
  return parent;
  }

  static final class MapSubscriber<T, R> extends Subscriber<T> {

  final Subscriber<? super R> actual;

  final Func1<? super T, ? extends R> mapper;

  boolean done;

  public MapSubscriber(Subscriber<? super R> actual, Func1<? super T, ? extends R> mapper) {
  this.actual = actual;
  this.mapper = mapper;
  }

  @Override
  public void onNext(T t) {
  R result;

  try {
  // 订阅的时候 自上而下， 先收到原类型T的事件，然后通过mapper即上面的func1，转化为需要的R类型结果，
  // 这样目标Subscriber就可以处理了
  result = mapper.call(t);
  } catch (Throwable ex) {
  Exceptions.throwIfFatal(ex);
  unsubscribe();

  onError(OnErrorThrowable.addValueAsLastCause(ex, t));
  return;
  }

  actual.onNext(result);
  }

```

其实就是生成了一个中间可以处理T类型事件的Subscriber ，收到事件后，通过func1转化为R类型事件，交给目标Subscriber 去处理。

其他转化，基本类似

![lift](liftflow.png)


### 线程切换

除了各种转化api，RxJava另一个吸引点就是方便的线程切换

常用如下：

```java
xxx.subscribeOn(Schedulers.io()) .observeOn(Schedulers.computation())
```
Schedulers.io()和Schedulers.computation()返回的都是Scheduler. Scheduler是负责执行任务的单元,而Schedulers是创建各种Scheduler的工厂.

####Scheduler种类

RxJava中有下面几种Scheduler:

调度器类型	说明
Schedulers.computation()	用于计算任务，如事件循环或和回调处理，不要用于IO操作(IO操作请使用Schedulers.io())；默认线程数等于处理器的数量
Schedulers.io()	用于IO密集型任务，如异步阻塞IO操作，这个调度器的线程池会根据需要增长；对于普通的计算任务，请使用Schedulers.computation()；Schedulers.io( )默认是一个CachedThreadScheduler，很像一个有线程缓存的新线程调度器
Schedulers.from(executor)	使用指定的Executor作为调度器
Schedulers.newThread()	为每个任务创建一个新线程
Schedulers.immediate()	在当前线程立即开始执行任务
Schedulers.trampoline()	当其它排队的任务完成后，在当前线程排队开始执行
Schedulers.immediate和Schedulers.trampoline都在当前线程执行,区别是前者马上执行,后者是等队列中的其他任务完成后才执行.

Scheduler是负责执行任务的工作单元.

它包含一个接口createWorker,负责创建Workder,以及一个包含一个静态的抽象类Worker.

```java
public abstract class Scheduler {

  /**
  * Retrieves or creates a new Scheduler.Worker} that represents serial execution of actions.
  * When work is completed it should be unsubscribed using {@link Scheduler.Worker#unsubscribe()}.
  * Work on a {@link Scheduler.Worker} is guaranteed to be sequential.
  *
  * @return a Worker representing a serial queue of actions to be executed
  */
  public abstract Worker createWorker();
  public abstract static class Worker implements Subscription {
  ...
  }
}
```
任务真正的执行是在Workder中进行的, 它包含了如下三个方法,用来执行任务:

```java
public abstract static class Worker implements Subscription {
  Schedules an Action for execution.
  public abstract Subscription schedule(Action0 action);
  public abstract Subscription schedule(final Action0 action, final long delayTime, final TimeUnit unit);
  public Subscription schedulePeriodically(final Action0 action, long initialDelay, long period, TimeUnit unit) {
}
```

* 第一个方法安排执行任务.
* 第二个方法一段延迟后执行任务.
* 第三个方法是周期性执行任务.

至于Worker是怎么执行任务的,线程/线程池模型是什么样的,由各个不同的Worker决定.

先不管什么io等scheduler具体怎么实现，其实切换线程也是一种转化，也是生成新的observable，和lift也是差不多的，只是转化的过程不一样，先看下切换线程subscribeOn的代码

```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
//ScalarSynchronousObservable是一个用于发送单个常量值的Observable，就是我们平常调用just时只传一个参数时声称的observable

  if (this instanceof ScalarSynchronousObservable) {
  return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
  }
  return create(new OperatorSubscribeOn<T>(this, scheduler));
  }

  ```
这里就只对通用情况说明下，create不用多说，基本分析的就是生成的OperatorSubscribeOn 的call方法了

```java
@Override
  public void call(final Subscriber<? super T> subscriber) {
  // 可以看到就是通过传入的scheduler生成一个worker，比如我们常用的io等

  final Worker inner = scheduler.createWorker();
  subscriber.add(inner);

  // 然后在生成的worker里执行一个任务，在该任务里会调用消费者的 onStart onNext onCompleted
  inner.schedule(new Action0() {
  @Override
  public void call() {
  final Thread t = Thread.currentThread();

  Subscriber<T> s = new Subscriber<T>(subscriber) {
  @Override
  public void onNext(T t) {
  subscriber.onNext(t);
  }

  @Override
  public void onError(Throwable e) {
  try {
  subscriber.onError(e);
  } finally {
  inner.unsubscribe();
  }
  }

  @Override
  public void onCompleted() {
  try {
  subscriber.onCompleted();
  } finally {
  inner.unsubscribe();
  }
  }

  // Producer 生产者（用来处理反压），被观察和观察者之间的请求通道
  @Override
  public void setProducer(final Producer p) {
  subscriber.setProducer(new Producer() {
  @Override
  public void request(final long n) {
  if (t == Thread.currentThread()) {
  p.request(n);
  } else {
  inner.schedule(new Action0() {
  @Override
  public void call() {
  p.request(n);
  }
  });
  }
  }
  });
  }
  };

  source.unsafeSubscribe(s);
  }
  });
  }
  ```
简单说就是通过scheduler生成worker，在worker里生成一个Subscriber（调用该Subscriber时最终会调用到最后的目标Subscriber），source就是切换线程转化前的observable，然后通过source来调用订阅者
unsafeSubscribe代码如下：

```java
public final Subscription unsafeSubscribe(Subscriber<? super T> subscriber) {
  ...
  // new Subscriber so onStart it
  subscriber.onStart();
  // allow the hook to intercept and/or decorate
  hook.onSubscribeStart(this, onSubscribe).call(subscriber);
  return hook.onSubscribeReturn(subscriber);

  }
  ```
  不多说了，这就是和最初一样的了

observeOn(Scheduler scheduler)也基本类似,都是转化，但不是直接new一个onsubscribe，而是通过lift转化，就跟之前的差不多，只是对应不同的operate。

```java
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
  if (this instanceof ScalarSynchronousObservable) {
  return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
  }
  return lift(new OperatorObserveOn<T>(scheduler, delayError, bufferSize));
  }
```

### Backpressure

背压是指在异步场景中，被观察者发送事件速度远快于观察者的处理速度的情况下，一种告诉上游的被观察者降低发送速度的策略

简而言之，背压是流速控制的一种策略。

上面出现的Producer 就是用来处理Backpressure 的

当生产过快，而消费不过来时，就会发生异常 MissingBackpressureException

比如：

```java
Observable.interval(1, TimeUnit.MILLISECONDS)
  .observeOn(Schedulers.newThread())
  .subscribe(new Subscriber<Long>() {
  @Override
  public void onCompleted() {

  }

  @Override
  public void onError(Throwable e) {
  printLog("onerror:"+e.getClass());
  }

  @Override
  public void onNext(Long aLong) {
  try {
  Thread.sleep(100);
  } catch (Exception e) { }
  printLog("onnext:" + aLong);
  }
  });

```
会输出：
```java
14:43:34.056 31098-2164/com.example.wp.weeklyrxjavademo D/rxjava_test: onnext:0
05-12 14:43:34.056 31098-2164/com.example.wp.weeklyrxjavademo D/rxjava_test: onerror:class rx.exceptions.MissingBackpressureException
```

可以选用支持backpressure的api来避免这个问题，当一个请求完成后再去请求下一个，比如range, 然后再请求完一个再去请求下一个

```java
Observable.range(1, 10000)
  .observeOn(Schedulers.newThread())
  .subscribe(new Subscriber<Integer>() {
  @Override
  public void onStart() {
  request(1);
  }

  @Override
  public void onCompleted() {

  }

  @Override
  public void onError(Throwable e) {
  printLog("onerror:"+e.getClass());
  }

  @Override
  public void onNext(Integer aLong) {
  try {
  Thread.sleep(100);
  } catch (Exception e) { }
  printLog("onnext:" + aLong);
  request(1);
  }
  });

  }

```
request(1)代表拉取下一个，会调用producer的request去获取，如果将request的参数改为Long.MAX_VALUE，那就跟一般使用一样，相当于没有响应式拉取
其实去掉request（1），也是一样的，因为observeon内部就是用request去实现响应式拉取,在代码ObserveOnSubscriber里onnext->scheule->call

```java
public void call() {
...
if (currentEmission == limit) {
  requestAmount = BackpressureUtils.produced(requested, currentEmission);
  request(currentEmission);
  currentEmission = 0L;
  }
...
}

```

和backpressure相关的api还有很多，比如onBackpressurebuffer，onBackpressureDrop

不过一般该问题很少发生，而我们项目更加基本没有这样的场景

#### RxJava2.0

Flowable与Observable
在Rxjava1 中，有的Observable支持backpressure，有的不支持，在2里，对该情况进行了区分，Observable不再支持backpresuure，改为Flowable支持

Single、Completable
Single 与 Completable 都基于新的 Reactive Streams 的思想重新设计了接口，主要是消费者的接口， 现在他们是这样的：

```java
interface SingleObserver<T> {  
  void onSubscribe(Disposable d);
  void onSuccess(T value);
  void onError(Throwable error);
}

interface CompletableObserver<T> {  
  void onSubscribe(Disposable d);
  void onComplete();
  void onError(Throwable error);
}
```
