### 关于Hook

RxJava2提供了RxJavaPlugins这个类用来在做runtime hook。

#### Error handling

```
/**
 * Sets the specific hook function.
 * @param handler the hook function to set, null allowed
 */
public static void setErrorHandler(@Nullable Consumer<? super Throwable> handler) {
    if (lockdown) {
        throw new IllegalStateException("Plugins can't be changed anymore");
    }
    errorHandler = handler;
}
```

#### 在每次Observable创建时打印log

因为每个Observable创建都会走onAssembly这个方法，所以利用onObservableAssembly这个变量来hook

```java
@SuppressWarnings("rawtypes")
@Nullable
static volatile Function<? super Observable, ? extends Observable> onObservableAssembly;

/**
 * Calls the associated hook function.
 * @param <T> the value type
 * @param source the hook's input value
 * @return the value returned by the hook
 */
@SuppressWarnings({ "rawtypes", "unchecked" })
@NonNull
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```

hook代码如下

```java
RxJavaPlugins.setOnObservableAssembly(new Function<Observable, Observable>() {
    @Override
    public Observable apply(@NonNull final Observable observable) throws Exception {
        Log.e("yj", "---hook--" + observable.toString());
        return observable;
    }
});
```

1. RxJavaPlugins有很多提供hook的方法，我有个邪恶的想法是不是可以搞个很隐蔽的的bug呢
2. 后面分析源码时，**遇到RxJavaPlugins.onAssembly(source)这样的类似代码可直接忽略内部实现，直接返回source即可**

### 关于Observable

```
public abstract class Observable<T> implements ObservableSource<T> {
    public final void subscribe(Observer<? super T> observer) {
        ObjectHelper.requireNonNull(observer, "observer is null");
        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);

            ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");

            subscribeActual(observer);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            // can't call onError because no way to know if a Disposable has been set or not
            // can't call onSubscribe because the call might have set a Subscription already
            RxJavaPlugins.onError(e);

            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(e);
            throw npe;
        }
    }
    protected abstract void subscribeActual(Observer<? super T> observer);
}
```

1. Observable是一个抽象类，T为要发射的数据，ObservableSource接口只有一个subscribe方法
2. **subscribe的真正实现在subscribeActual方法里**
3. **一堆的静态方法（也就Observable的操作符）返回各种功能的Observable（根据具体的功能实现subscribeActual）**

### 简单的例子（create）

下面的代码用create创建一个Observable发送1、2、complete事件，并且不存在Schedulers调度。然后我们看下源码怎么实现的

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
        e.onNext(1);
        e.onNext(2);
        e.onComplete();
    }
}).subscribe(new Observer<Integer>() {
    @Override
    public void onSubscribe(@NonNull Disposable d) {
        Log.e("yj", "---onSubscribe==" + d.isDisposed());
    }

    @Override
    public void onNext(@NonNull Integer s) {
        Log.e("yj", "---onNext==" + s);

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

1. 首先new了一个响应式接口ObservableOnSubscribe，传给了Observable.create

2. Observable.create创建了一个Observable也就是ObservableCreate，那么当调用subscribe时直接看ObservableCreate的subscribeActual方法

   ~~~java
   @Override
   protected void subscribeActual(Observer<? super T> observer) {
       CreateEmitter<T> parent = new CreateEmitter<T>(observer);
       observer.onSubscribe(parent);

       try {
           source.subscribe(parent);
       } catch (Throwable ex) {
           Exceptions.throwIfFatal(ex);
           parent.onError(ex);
       }
   }
   ~~~

3. CreateEmitter<T> parent = new CreateEmitter<T>(observer);首先传入observer创建了一个CreateEmitter。CreateEmitter实现了ObservableEmitter和Disposable接口，源码先暂时不管简单的说就是对事件的emit和dispose（我直译为处理，就是类似Rxjava里的unsubscribed）

4. observer.onSubscribe(parent);传入parent也就是CreateEmitter，走**onSubscribe回调**

5. source.subscribe(parent);然后走source（也就是ObservableCreate传入的ObservableOnSubscribe）的**subscribe**方法。

6. 调用e.onNext(1);也就是调用CreateEmitter的onNext方法，先判断isDisposed，然后回调observer的**onNext**回调

   ```java
   @Override
   public void onNext(T t) {
       if (t == null) {
           onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
           return;
       }
       if (!isDisposed()) {
           observer.onNext(t);
       }
   }
   ```

7. 调用e.onComplete();也就是调用CreateEmitter的onComplete方法，走observer的**onComplete**回调,并且**状态置为dispose**

   ~~~java
   @Override
   public void onComplete() {
       if (!isDisposed()) {
           try {
               observer.onComplete();
           } finally {
               dispose();
           }
       }
   }
   ~~~

8. 再看onError：

   ```
   @Override
   public void onError(Throwable t) {
       if (!tryOnError(t)) {
           RxJavaPlugins.onError(t);
       }
   }

   @Override
   public boolean tryOnError(Throwable t) {
       if (t == null) {
           t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
       }
       if (!isDisposed()) {
           try {
               observer.onError(t);
           } finally {
               dispose();
           }
           return true;
       }
       return false;
   }
   ```

   1. 首先判断dispose状态，如果是dispose状态则tryOnError返回fasle，抛出异常crash之
   2. isDispose状态为false时，走**observer.onError回调**，同时**状态置为dispose**

9. 从next、error、complete代码分析，我们可以得知

   1. **一旦走了onError或者onComplete，都会导致事件流被打断（状态为dispose）**
   2. **一旦状态为dispose，后面的onNext、onComplete、onError事件都无法接收；同时如果发送onError还会导致crash**

### 关于Disposable

控制resource的disposable状态的接口

```java
public interface Disposable {
    /**
     * Dispose the resource, the operation should be idempotent.
     */
    void dispose();

    /**
     * Returns true if this resource has been disposed.
     * @return true if this resource has been disposed
     */
    boolean isDisposed();
}
```

源码里Disposable接口的实现类基本都会继承AtomicReference<Disposable>的原子类，类似下面的代码

```java
final class CreateEmitter<T> extends AtomicReference<Disposable> implements Disposable
```

拿之前的CreateEmitter为例分析，分别看下面的三个方法，都会用到DisposableHelper

```
static final class CreateEmitter<T>
extends AtomicReference<Disposable>
implements ObservableEmitter<T>, Disposable {
        @Override
        public void setDisposable(Disposable d) {
            DisposableHelper.set(this, d);
        }
        
        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }
        
        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
}
```

#### isDisposed

```
public enum DisposableHelper implements Disposable {
    /**
     * The singleton instance representing a terminal, disposed state, don't leak it.
     */
    DISPOSED
    ;

    /**
     * Checks if the given Disposable is the common {@link #DISPOSED} enum value.
     * @param d the disposable to check
     * @return true if d is {@link #DISPOSED}
     */
    public static boolean isDisposed(Disposable d) {
        return d == DISPOSED;
    }
```

1. 这里有一个用enum实现的Disposable对象的单例**DISPOSED**（因为是enum实现的所以能保证线程安全）
2. isDisposed直接判断AtomicReference的get()得到的对象是否和DISPOSED相等；这里因为创建CreateEmitter时没有调用过set，所以一开始肯定是null

#### dispose

```java
/**
 * Atomically disposes the Disposable in the field if not already disposed.
 * @param field the target field
 * @return true if the current thread managed to dispose the Disposable
 */
public static boolean dispose(AtomicReference<Disposable> field) {
    Disposable current = field.get();// current获取当前field的value值
    Disposable d = DISPOSED;
    if (current != d) {
        // 如果不是DISPOSED状态
        current = field.getAndSet(d);// field设置为DISPOSED，并且返回旧的值给current
        if (current != d) {
        // 判断旧的值是否为DISPOSED，如果不为DISPOSED则返回true
            if (current != null) {
                current.dispose();// 调用旧的value的dispose方法，这里不知道为啥要这样做，感觉不用也可以
            }
            return true;
        }
    }
    // 如果field之前就是DISPOSED状态，返回false
    return false;
}
```

1. 改变状态为DISPOSED，如果之前状态不为DISPOSED返回true，否则返回false
2. 这里更改状态，做了个double check，并且保证线程安全

#### setDisposable

```java
/**
 * Atomically sets the field and disposes the old contents.
 * @param field the target field
 * @param d the new Disposable to set
 * @return true if successful, false if the field contains the {@link #DISPOSED} instance.
 */
public static boolean set(AtomicReference<Disposable> field, Disposable d) {
    for (;;) {
        Disposable current = field.get();
        if (current == DISPOSED) {
        // 如果当前field是DISPOSED状态直接返回false
            if (d != null) {
                d.dispose();
            }
            return false;
        }
        if (field.compareAndSet(current, d)) {
            // 如果当前field的value和current一样，也就是field没有变化，则更新Disposable为d，并且返回true
            // 否则继续走for循环
            if (current != null) {
                current.dispose();
            }
            return true;
        }
    }
}
```

1. 这里根据CAS原理，利用一个死循环和compareAndSet来实现更新Disposable，保证线程安全
2. 如果field的状态为DISPOSED，则直接返回false不能更新；否则更新成功，返回true
3. 那么这里为什么不直接用getAndSet呢？因为要判断DISPOSED状态返回false，如果直接getAndSet就有可能在DISPOSED状态下更新导致出错

