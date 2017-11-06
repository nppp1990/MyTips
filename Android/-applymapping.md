

### -applymapping

proguard配置中可以加上这样一行，这个mapping.txt是在app目录下的一个mapping文件，那么applymapping有什么功能呢，我们先做个试验

```java
-applymapping mapping.txt
```

MainActivity代码如下

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        test("-----");
    }
    
    private void test(String s) {
        System.out.print(s);
    }

}
```

最终在app/build/outputs/mapping/release下得到的mapping对应如下**test函数对应的mapping是“a”**

```
com.xmtj.testproguard.MainActivity -> com.xmtj.testproguard.MainActivity:
    void <init>() -> <init>
    void onCreate(android.os.Bundle) -> onCreate
    void test(java.lang.String) -> a
```



接着我们修改MainActivity，在test函数上加一个test1

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        test("-----");
        test2("-----");
    }

    private void test2(String s) {
        System.out.print(s);
    }
    
    private void test(String s) {
        System.out.print(s);
    }

}
```

最终得到的mapping如下，可以看到**test函数变成了b，test2为a**

```
com.xmtj.testproguard.MainActivity -> com.xmtj.testproguard.MainActivity:
    void <init>() -> <init>
    void onCreate(android.os.Bundle) -> onCreate
    void test2(java.lang.String) -> a
    void test(java.lang.String) -> b
```

只是在test函数上面加了一个test2函数，却导致了test的mapping发生了变化，那么我们有什么办法解决呢？这里applymapping的作用就来了。

### applymapping的用途

1. 将第一次编译产生的app/build/outputs/mapping/release/mapping.txt文件拷贝到app目录下，这里为了区分重命名为mapping1.txt（也可以不改名字）
2. proguard配置加上-applymapping mapping1.txt

最终得到的mapping文件如下，可以看到**test函数没有变成b**

```
com.xmtj.testproguard.MainActivity -> com.xmtj.testproguard.MainActivity:
    void <init>() -> <init>
    void onCreate(android.os.Bundle) -> onCreate
    void test2(java.lang.String) -> b
    void test(java.lang.String) -> a
```

