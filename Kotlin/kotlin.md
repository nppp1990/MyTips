Kotlin

#### startActivity

```java
Intent(this, DayActivity::class.java).apply {
    putExtra(DAY_CODE, intent.getStringExtra(DAY_CODE))
    startActivity(this)
}

/**
 * Calls the specified function [block] with `this` value as its receiver and returns `this` value.
 */
@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T { block(); return this }
```

 这里apply函数一般用于统一初始化或者赋值某一个变量，看起来更加清晰



#### Unit

```
/**
 * The type with only one value: the Unit object. This type corresponds to the `void` type in Java.
 */
public object Unit {
    override fun toString() = "kotlin.Unit"
}
```



#### When

```java
when {
    a == 1 -> {

    }

    c == 2 -> {

    }
}
var b = when (a) {
    1 -> {
        2
    }
    2 -> {
        3
    }
    else -> {
        4
    }
}
```

1. 类似于switch，正常的逻辑判断
2. when(a) { } 或者when{ }
3. 可以作为表达式，有返回值

### fun

1. 定义：控制符修饰+fun+方法名+参数列表+返回值类型 + 函数体

~~~java
fun double(x: Int): Int {
    return 2*x
}
~~~

2. 作为表达式返回；实例调用

3. 函数体也可以为表达式

   ```kotlin
   fun sum(a: Int, b: Int) = a + b
   ```

4. 默认参数:使用时可以不传则用默认参数的value

5. 多参数：vararg

6. 本地函数：嵌套函数，Kotlin supports local functions, i.e. a function inside another function:

   ~~~kotlin
   fun dfs(graph: Graph) {
       fun dfs(current: Vertex, visited: Set<Vertex>) {
           if (!visited.add(current)) return
           for (v in current.neighbors)
               dfs(v, visited)
       }

       dfs(graph.vertices[0], HashSet())
   }
   ~~~

7. 泛型函数:同java一样

   ~~~kotlin
   fun <T> singletonList(item: T): List<T> {
       // ...
   }
   ~~~

   ​

#### 可见性修饰符

public：

internal：同模块

protected：private+子类

private：仅在当前类（不同于java，外部类不能访问内部类的private对象）

#### 艾维斯操作符(Elvis Operator)

如果?:左边的表达式不为空则返回它，否则返回右边的

~~~kotlin
val l =b?.length ?: -1
~~~

#### companion

1. static
2. only one default static class

#### lateinit

1. 表示延迟加载的变量 必须用var修饰不能用val
2. 如果在初始化之前访问该变量会抛出异常lateinit property mPagerDays has not been initialized

#### !!

如果被!!修饰的对象在访问时为空会抛出异常kotlin.KotlinNullPointerException

#### object

1. 声明一个单例，lazy-init
2. 没有构造函数

#### 单例(双侧校验)

~~~kotlin
class Singleton private constructor(str:String) {
    init {
        // do init
    }
    compaion object {
        @Volatile
        var singleton: Singleton? = null
        fun getInstance(str: String): Singleton {
            if (singleton == null) {
                synchronized(Singleton::class) {
                    if (singleton == null) {
                        singleton = Singleton(str)
                    }
                }
            }
            return singleton!!
        }
    }
}
~~~



#### 函数回调作为参数

~~~kotlin
var callback :(()->Unit)

//调用的的地方
callback.invoke()或者callback()
//传入的地方
{
  //doSomeThing
}
~~~

#### as

1. 重命名

   ~~~kotlin
   import net.mindview.simple.Vecotr
   import java.until.Vecotr as aVector

   val v : Vecotr = Vector()
   val vA : aVector = aVector()
   ~~~

2. 类型转换

#### 文件类型(Android Studio对应的)

New----Kotlin File/Class对应下面五个选项，实际上都是.kt文件

1. File：存放常量，工具方法（extension）
2. Class：类
3. Interface：接口
4. Enum：枚举
5. object：单例,工具方法（util）

#### Class Inheritance

1. 默认是final的，不能被继承
2. open或者abstract
3. ​

#### companion object

相当于静态类companion object A（A为静态内部类名，可以省略）

#### Any

所有类的默认父类

#### data class

1. 构造函数必须至少有一个参数
2. 构造函数的参数必须指定var或者val
3. 不能继承，但是可以实现接口
4. 不能是abstract, open, sealed or inner

#### @Volatile

相当于java的volatile

#### JvmPlatformAnnotations.kt

#### kotlin转java

Tools>Kotlin>Show Kotlin Bytecode>Decompile 虽然不是比较美观的java代码，可以作参考研究Kotlin怎么实现的

#### 匿名内部类

~~~kotlin
        recycler_view.addOnScrollListener(object : RecyclerView.OnScrollListener() {
            override fun onScrolled(recyclerView: RecyclerView?, dx: Int, dy: Int) {
               //
            }
        })
~~~

1. object:

#### 位运算

1. and
2. or
3. xor