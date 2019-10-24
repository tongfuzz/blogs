###Kotlin 知识总结

* 如果主构造函数没有任何注解或者可见性修饰符，可以省略这个 constructor 关键字，主构造函数不能包含任何的代码。初始化的代码可以放到以 init 关键字作为前缀的初始化块（initializer blocks）中
* reified 将内联函数的类型参数标记为在运行时可访问
* kotlin中的in和out

>如果你的类是将泛型作为内部方法的返回，那么可以用 out,如果你的类是将泛型对象作为函数的参数，那么可以用 in,<font color=red>对于这两个暂时没有理解清楚，只能暂时死记硬背为如果使用 ？extend 可以使用out ，如果使用 ？super 可以使用in</font>如下所示

```java
interface Production<out T> {
    fun produce(): T
}

interface Consumer<in T> {
    fun consume(item: T)
}
```

* 对于使用@Jvmstatic注解声明的字段和方法，可以通过类名称直接调用，
* Kotlin 的可见性以下列方式映射到 Java：

>private 成员编译成 private 成员</br>
private 的顶层声明编译成包级局部声明；</br>
protected 保持 protected（注意 Java 允许访问同一个包中其他类的受保护成员， 而 Kotlin 不能，所以 Java 类会访问更广泛的代码）；</br>
internal 声明会成为 Java 中的 public。internal 类的成员会通过名字修饰，使其更难以在 Java 中意外使用到，并且根据 Kotlin 规则使其允许重载相同签名的成员而互不可见；</br>
public 保持 public。

* 关于@JvmSuppressWildcards和@JvmWildcard注解的说明<br/>

>如果我们在默认不生成通配符的地方需要通配符，我们可以使用 @JvmWildcard 注解：

```java
fun boxDerived(value: Derived): Box<@JvmWildcard Derived> = Box(value)
// 将被转换成
// Box<? extends Derived> boxDerived(Derived value) { …… }
```
>另一方面，如果我们根本不需要默认的通配符转换，我们可以使用@JvmSuppressWildcards

```java
fun unboxBase(box: Box<@JvmSuppressWildcards Base>): Base = box.value
// 会翻译成
// Base unboxBase(Box<Base> box) { …… }
```
>@JvmSuppressWildcards 不仅可用于单个类型参数，还可用于整个声明（如函数或类），从而抑制其中的所有通配符


* kotlin 中的foreach 可以使用for(article in List)来实现

```java
for(article in data){
  article.top=true
}
```

* kotlin 中value和postValue的区别，value在主线程调用，postValue在其他线程调用