###Dagger2知识总结

* 关机多重绑定的使用

  >dagger允许绑定一些对象到集合中，哪怕这些绑定存在于不同的module中，我们可以使用多重绑定在不同的module中绑定对象，并在某一个类中使用这个绑定对象,具体使用详见[多重绑定的使用](https://dagger.dev/multibindings.html)
 
一般我们使用多重绑定生成的对象，我们可以使用@Inject 直接生成它的对象，也可以作为可注入对象的参数，<font color=red>注意，使用对象是可能需要声明@JvmSuppressWildcards来去除通配符</font>，如下：

```java
//声明多重绑定对象
@Binds
@IntoMap
@StringKey("haha")
internal abstract fun bindStringInfo(viewModel:HomePageViewModel):ViewModel

//使用多重绑定对象
class NavigationViewModel @Inject constructor(map:@JvmSuppressWildcards Map<String,ViewModel>) : ViewModel() {
    init {
        println("haha"+map.size)
    }

}


```
  
  