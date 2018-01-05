### subscribeOn

1. 找到ObservableSubscribeOn类

   ```java
   public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
       super(source);
       // 传入上游的Observable和调度器Scheduler
       this.scheduler = scheduler;
   }

   @Override
   public void subscribeActual(final Observer<? super T> s) {
       // 用下游的observer创建一个新的observer
       final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

       s.onSubscribe(parent);
       // scheduler直接执行SubscribeTask
       parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
   }
   ```

2. 查看SubscribeOnObserver类，除了实现Disposable接口还实现了Observer接口，说明他还是个新的observer；

   ```
   static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

       private static final long serialVersionUID = 8094547886072529208L;
       final Observer<? super T> actual;

       final AtomicReference<Disposable> s;

       SubscribeOnObserver(Observer<? super T> actual) {
           this.actual = actual;// 下游的observer
           this.s = new AtomicReference<Disposable>();
       }

       @Override
       public void onSubscribe(Disposable s) {
           // 上游的onSubscribe会调用，但是因为this.s的disposable不为null，大部分情况一直都是直接跳过
           DisposableHelper.setOnce(this.s, s);
       }

       @Override
       public void onNext(T t) {
           actual.onNext(t);
       }

       @Override
       public void onError(Throwable t) {
           actual.onError(t);
       }

       @Override
       public void onComplete() {
           actual.onComplete();
       }

       @Override
       public void dispose() {
           DisposableHelper.dispose(s); // 这里dispose多了个步骤，没明白？？？？
           DisposableHelper.dispose(this);
       }

       @Override
       public boolean isDisposed() {
           return DisposableHelper.isDisposed(get());
       }

       void setDisposable(Disposable d) {
           DisposableHelper.setOnce(this, d);
       }
   }
   ```

3. 查看SubscribeTask，这是个Runnable，并且是ObservableSubscribeOn的内部类

   ~~~java
   final class SubscribeTask implements Runnable {
       private final SubscribeOnObserver<T> parent;

       SubscribeTask(SubscribeOnObserver<T> parent) {
           this.parent = parent;
       }

       @Override
       public void run() {
           // 上游的observable执行subscribe，传入的是下游的obser
           source.subscribe(parent);
       }
   }
   ~~~

4. 查看scheduler.scheduleDirect，这个方法就是用scheduler立即执行传入的Runnable任务

   ```java
   @NonNull
   public Disposable scheduleDirect(@NonNull Runnable run) {
       return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
   }
   ```

5. 所以总结下

   1. ObservableSubscribeOn起到一个桥接的功能，执行**source.subscribe(parent)**，处理上游的Observable
   2. 传入的scheduler用来控制怎么执行

### observeOn

1. 找到ObservableObserveOn类

   ```java
   public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
       final Scheduler scheduler;
       final boolean delayError;
       final int bufferSize;
       public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
           super(source);
           this.scheduler = scheduler; // 调度器
           this.delayError = delayError;// 
           this.bufferSize = bufferSize;// 任务队列大小，默认128
       }

       @Override
       protected void subscribeActual(Observer<? super T> observer) {
           if (scheduler instanceof TrampolineScheduler) {
               // 如果是TrampolineScheduler，放个屁啥都不干？
               source.subscribe(observer);
           } else {
               // scheduler.createWorker
               Scheduler.Worker w = scheduler.createWorker();
               // 下游的observer包装成ObserveOnObserver，传给上游的source.subscribe
               source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
           }
       }
   ```

2. 查看ObserveOnObserver类:AtomicInteger的子类，实现了QueueDisposable，Observer<T>, Runnable

   ```java
   static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
   implements Observer<T>, Runnable {

       private static final long serialVersionUID = 6576896619930983584L;
       final Observer<? super T> actual;
       final Scheduler.Worker worker;
       final boolean delayError;
       final int bufferSize;

       SimpleQueue<T> queue; // 一个队列

       Disposable s;

       Throwable error;
       volatile boolean done;// 标记是否结束，和disposable不一样的是如果done为true，还是会走OnComplete或者onError
       volatile boolean cancelled;// 标记disposable状态

       int sourceMode;

       boolean outputFused;

       ObserveOnObserver(Observer<? super T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
           this.actual = actual;
           this.worker = worker;
           this.delayError = delayError;
           this.bufferSize = bufferSize;
       }
   ```

3. 查看ObserveOnObserver类的onSubscribe方法

   ```java
   @Override
   public void onSubscribe(Disposable s) {
       if (DisposableHelper.validate(this.s, s)) {
           this.s = s;
   // 这里省略了一部分FuseMode的代码，默认情况不会走
           // 初始化队列
           queue = new SpscLinkedArrayQueue<T>(bufferSize);
           // 下游observer的onSubscribe回调
           actual.onSubscribe(this);
       }
   }
   ```

4. 查看disposable的逻辑，cancelled变量标记，dispose同时还会清空队列和dispose任务worker

   ```java
   @Override
   public void dispose() {
       if (!cancelled) {
           cancelled = true;
           s.dispose();
           worker.dispose();// 中断Scheduler任务
           if (getAndIncrement() == 0) {
               // 清空队列
               queue.clear();
           }
       }
   }

   @Override
   public boolean isDisposed() {
       return cancelled;
   }
   ```

5. 查看ObserveOnObserver的onNext，把数据塞到队列里，并且只有AtomicInteger值为0，才执行任务worker.schedule

   ```java
   @Override
   public void onNext(T t) {
       if (done) {
           return;
       }

       if (sourceMode != QueueDisposable.ASYNC) {
           // 入队列
           queue.offer(t);
       }
       schedule();
   }

   void schedule() {
       if (getAndIncrement() == 0) {
           // AtomicInteger的值为0就执行worker.schedule，传入的this是Runnable；AtomicInteger加1
           worker.schedule(this);
       }
   }
   ```

6. ObserveOnObserver的Runnable实现run方法

   ```java
   @Override
   public void run() {
       if (outputFused) {
           // 默认为false，先不管
           drainFused();
       } else {
           drainNormal();
       }
   }

   void drainNormal() {
       int missed = 1;// 这里成功接收一次数据missed就加1，默认为1

       final SimpleQueue<T> q = queue;
       final Observer<? super T> a = actual;

       for (;;) {
           if (checkTerminated(done, q.isEmpty(), a)) {
               // disposable或者done的情况会返回true
               return;
           }

           for (;;) {
               boolean d = done;
               T v;

               try {
                   v = q.poll();// 取出队列一个数据v
               } catch (Throwable ex) {
                   Exceptions.throwIfFatal(ex);
                   s.dispose();
                   q.clear();
                   a.onError(ex);
                   worker.dispose();
                   return;
               }
               boolean empty = v == null;

               if (checkTerminated(d, empty, a)) {
                   // 再次检查是否已经done或者diposable
                   return;
               }

               if (empty) {
                   // 如果队列空了跳出里面的for循环
                   break;
               }
               // 下游的Observer回调onNext
               a.onNext(v);
           }

           // 更新missed值，表示还剩几个走了schedule但是还没有被调用onNext的任务
           missed = addAndGet(-missed);
           if (missed == 0) {
               // missed为0跳出for循环
               break;
           }
       }
   }
   ```

   这里解释下为什么要两个for循环和为什么用AtomicInteger标记schedule次数，因为要考虑上游事件发送和下游事件接受速度是不一样，而且worker.schedule导致上下游不在一个线程，比如下面几个例子

   1. 发送数据（1，2，3，4）很快，接收数据很慢：那么异步情况下，很快的会调用四次schedule，getAndIncrement只有第一次为0，只会走一次worker.schedule(this)，那么run方法就只会走一次，会在外面的for循环跳出，也就是missed=0结束
   2. 发送数据（1，2，3，4，complete）很快，接收数据很慢：基本同上，但是因为发送了complete导致状态为disposable，会在里面的for循环return，因为checkTerminated(d, empty, a)返回了true
   3. 发送数据（1，2）很慢，发送数据（3，4）很快，接收数据（2）很慢：那么异步情况下，有可能在2执行完onNext，刚刚跳出里面的for循环，这时候发送数据3了导致队列不为空，miss不为空了所以不会跳出外面的for循环。这样就不用worker.schedule(this);

7. 总结下

   1. 同样是桥接，这里对下游的observer做处理
   2. 传入的scheduler用来控制怎么执行

### Example

下面代码是我们很常见的一个例子，发送数据（1，2）在IO线程执行，Observer在UI线程中执行

```java
Observable.just(1,2)
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                Log.e("yj", "---onSubscribe==" + d.isDisposed());
            }

            @Override
            public void onNext(@NonNull Integer i) {
                Log.e("yj", "---onNext==" + i);
            }

            @Override
            public void onError(@NonNull Throwable e) {
                Log.e("yj", "---onError==" + e);
            }

            @Override
            public void onComplete() {
                Log.e("yj", "---onComplete");
            }
        });
```

1. 依次创建了三个Observable：ObservableFromArray，ObservableSubscribeOn，ObservableObserveOn
2. 从下往上看subscribeActual方法调用：ObservableObserveOn不做处理，ObservableSubscribeOn使subscribe在io线程中执行，ObservableFromArray顺序发送1，2
3. Observer的回调只有ObservableObserveOn处理了，使其在UI线程中被调用

### 举个的例子

下面的代码执行了两次subscribeOn和observeOn，那么just和accept在哪个线程执行呢?

```
Observable.just(1,2)
        .subscribeOn(Schedulers.io())
        .subscribeOn(AndroidSchedulers.mainThread())
        .observeOn(AndroidSchedulers.mainThread())
        .observeOn(Schedulers.io())
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(@NonNull Integer integer) throws Exception {
                
            }
        });
```

答案：在两个不同的IO线程

