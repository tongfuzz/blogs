###Kotlin 知识总结

* 如果主构造函数没有任何注解或者可见性修饰符，可以省略这个 constructor 关键字，主构造函数不能包含任何的代码。初始化的代码可以放到以 init 关键字作为前缀的初始化块（initializer blocks）中
* reified 将内联函数的类型参数标记为在运行时可访问
* kotlin中的in和out

>如果你的类是将泛型作为内部方法的返回，那么可以用 out,如果你的类是将泛型对象作为函数的参数，那么可以用 in,如下所示

```java
interface Production<out T> {
    fun produce(): T
}

interface Consumer<in T> {
    fun consume(item: T)
}
```

* 