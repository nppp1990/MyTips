### 阅读Observable的xxx操作符的步骤

1. 找到Observable的子类ObservableXXX

   ```java
   RxJavaPlugins.onAssembly(new ObservableXXX<T>());
   ```

2. 查看ObservableXXX的subscribeActual(Observer<? super T> s)函数，一般做下面三件事

   1. 一般会创建一个Disposable接口的实现类d
   2. 调用s.onSubscribe(d);
   3. 具体subscribe实现代码

3. 具体subscribe实现代码需要关注的几个点

   1. disposed的实现
   2. observer的onNext，OnComplete，OnError何时被调用

### 以Observable.just为例

1. 找到ObservableFromArray

2. 查看ObservableFromArray的subscribeActual函数，发现主要逻辑都在FromArrayDisposable的run方法里

   ```java
   public final class ObservableFromArray<T> extends Observable<T> {
       final T[] array;
       public ObservableFromArray(T[] array) {
           this.array = array;
       }
       @Override
       public void subscribeActual(Observer<? super T> s) {
           FromArrayDisposable<T> d = new FromArrayDisposable<T>(s, array);

           s.onSubscribe(d);

           if (d.fusionMode) {
               // 根据fusion mode值得来的，具体看QueueFuseable，默认为false，暂时先不管 
               return;
           }

           d.run();
       }
   ```

3. 查看FromArrayDisposable的run

   ```java
   static final class FromArrayDisposable<T> extends BasicQueueDisposable<T> {

       final Observer<? super T> actual;

       final T[] array;

       int index;

       boolean fusionMode;

       volatile boolean disposed;

       FromArrayDisposable(Observer<? super T> actual, T[] array) {
           this.actual = actual;
           this.array = array;
       }

       @Override
       public void dispose() {
           // 用一个boolean变量来标记
           disposed = true;
       }

       @Override
       public boolean isDisposed() {
           return disposed;
       }

       void run() {
           T[] a = array;
           int n = a.length;

           for (int i = 0; i < n && !isDisposed(); i++) {
               // 如果没有Disposed则遍历array数组
               T value = a[i];
               if (value == null) {
                   // 如果value为null走error，跳出for循环
                   actual.onError(new NullPointerException("The " + i + "th element is null"));
                   return;
               }
               // 走onNext
               actual.onNext(value);
           }
           if (!isDisposed()) {
               // 如果没有Disposed，走onComplete
               actual.onComplete();
           }
       }
   }
   ```

4. 所以总结下：Just操作符依次发送数组中的事件，并且碰到null就中断；并且除非手动dispose状态一直都不会变

