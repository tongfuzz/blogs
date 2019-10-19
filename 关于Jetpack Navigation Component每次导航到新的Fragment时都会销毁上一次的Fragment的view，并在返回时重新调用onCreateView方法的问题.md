###关于Jetpack Navigation Component每次导航到新的Fragment时都会销毁上一次的Fragment的view，并在返回时重新调用onCreateView方法的问题

当使用Navigation组件时，由fragment1跳转到fragment2时会经历如下过程

* Fragment 1 created
* Fragment 1 view created (This contains the webview)
* Button clicked
* Fragment 1 view destroyed
* Fragment 2 created
* Fragment 2 view created
* Back clicked
* Fragment 1 view re-created
* Fragment 2 view destroyed
* Fragment 2 destroyed

每次的跳转都会销毁上一个fargment的view，目前google并没有给出官方的解决办法，只能通过将view对象保存到fragment中的方法，如下

```java
private var inflaterView: View? = null

override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?,savedInstanceState: Bundle?): View? {

        if (inflaterView == null) {
            inflaterView = inflater.inflate(R.layout.fragment_invite, null)
        }

        return inflaterView
    }
```

对于使用BottomNavigationView+Navigation做主页页面导航时也会遇到这个问题，但是对于这种情况google官方给出了目前暂时的解决办法具体参考官方提供的示例[NavigationAdvancedSample](https://github.com/android/architecture-components-samples/tree/master/NavigationAdvancedSample) 

相关issue讨论见[Support multiple back stacks for Bottom tab navigation](https://issuetracker.google.com/issues/80029773)