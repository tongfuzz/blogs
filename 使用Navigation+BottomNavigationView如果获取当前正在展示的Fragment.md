###使用Navigation+BottomNavigationView如果获取当前正在展示的Fragment

在使用Navigation+BottomNavigationView时，有时候我们需要Activity与Fragment进行通信，使用add,repalce等方法我们可以通过findViewById,findViewByTag等方法获取到当前fragment，但是使用Navigation我们获取fragment的方式发生了改变，我们需要通过如下方式来获取

```java
val primaryNavigationFragment = supportFragmentManager.primaryNavigationFragment
val fragments = primaryNavigationFragment?.childFragmentManager?.fragments
val fragment = fragments?.get(0)
```

详见[google issue](https://issuetracker.google.com/issues/119800853)


关于Navigation+BottomNavigationView的配合使用详见[Support multiple back stacks for Bottom tab navigation](https://issuetracker.google.com/issues/80029773)