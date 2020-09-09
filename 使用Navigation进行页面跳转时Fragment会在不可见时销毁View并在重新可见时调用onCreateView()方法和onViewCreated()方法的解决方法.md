##使用Navigation进行页面跳转时Fragment会在不可见时销毁View并在重新可见时调用onCreateView()方法和onViewCreated()方法的解决方法

在使用Navigation进行页面间的跳转时，比如我们从fragment1通过navigate方法跳转到fragment2时，fragment1会调用onDestoryView()方法销毁当前fragment的view，并在fragment2返回到fragment1时会
调用onCreateView()方法和onViewCreated()方法，这是因为在Navigation中Fragment之间的跳转方式最终是通过FragmentManager的replace方法来切换不同的fragment的，我们可以在FragmentNavigator的navigate方法中看到具体的逻辑：

```java
 @SuppressWarnings("deprecation") /* Using instantiateFragment for forward compatibility */
    @Nullable
    @Override
    public NavDestination navigate(@NonNull Destination destination, @Nullable Bundle args,
            @Nullable NavOptions navOptions, @Nullable Navigator.Extras navigatorExtras) {
        if (mFragmentManager.isStateSaved()) {
            Log.i(TAG, "Ignoring navigate() call: FragmentManager has already"
                    + " saved its state");
            return null;
        }
        String className = destination.getClassName();
        if (className.charAt(0) == '.') {
            className = mContext.getPackageName() + className;
        }
        final Fragment frag = instantiateFragment(mContext, mFragmentManager,
                className, args);
        frag.setArguments(args);
        //1:通过FragmentManager创建FragmentTransaction
        final FragmentTransaction ft = mFragmentManager.beginTransaction();
		  //2:设置fragent的进出动画
        int enterAnim = navOptions != null ? navOptions.getEnterAnim() : -1;
        int exitAnim = navOptions != null ? navOptions.getExitAnim() : -1;
        int popEnterAnim = navOptions != null ? navOptions.getPopEnterAnim() : -1;
        int popExitAnim = navOptions != null ? navOptions.getPopExitAnim() : -1;
        if (enterAnim != -1 || exitAnim != -1 || popEnterAnim != -1 || popExitAnim != -1) {
            enterAnim = enterAnim != -1 ? enterAnim : 0;
            exitAnim = exitAnim != -1 ? exitAnim : 0;
            popEnterAnim = popEnterAnim != -1 ? popEnterAnim : 0;
            popExitAnim = popExitAnim != -1 ? popExitAnim : 0;
            ft.setCustomAnimations(enterAnim, exitAnim, popEnterAnim, popExitAnim);
        }
		 //3:通过replace方法移除其他fragment，展示当前fragment
        ft.replace(mContainerId, frag);
        ft.setPrimaryNavigationFragment(frag);

        final @IdRes int destId = destination.getId();
        final boolean initialNavigation = mBackStack.isEmpty();
        // TODO Build first class singleTop behavior for fragments
        final boolean isSingleTopReplacement = navOptions != null && !initialNavigation
                && navOptions.shouldLaunchSingleTop()
                && mBackStack.peekLast() == destId;

        boolean isAdded;
        if (initialNavigation) {
            isAdded = true;
        } else if (isSingleTopReplacement) {
            // 4:如果fragment是singletop启动模式，
            if (mBackStack.size() > 1) {
                mFragmentManager.popBackStack(
                        generateBackStackName(mBackStack.size(), mBackStack.peekLast()),
                        FragmentManager.POP_BACK_STACK_INCLUSIVE);
                //5:添加fragment到fragmentmanager回退栈内
                ft.addToBackStack(generateBackStackName(mBackStack.size(), destId));
            }
            isAdded = false;
        } else {
        //5:添加fragment到fragmentmanager回退栈内
            ft.addToBackStack(generateBackStackName(mBackStack.size() + 1, destId));
            isAdded = true;
        }
        if (navigatorExtras instanceof Extras) {
            Extras extras = (Extras) navigatorExtras;
            for (Map.Entry<View, String> sharedElement : extras.getSharedElements().entrySet()) {
                ft.addSharedElement(sharedElement.getKey(), sharedElement.getValue());
            }
        }
        ft.setReorderingAllowed(true);
        ft.commit();
        // The commit succeeded, update our view of the world
        if (isAdded) {
            //如果不是singletop启动模式，将当前fragment添加到backstack列表中，以便在onSaveState()和onRestoreState(@Nullable Bundle savedState)方法中恢复状态
            mBackStack.add(destId);
            return destination;
        } else {
            return null;
        }
    }

```
在第三步调用replace方法相当于调用了remove方法移除了所有当前有相同containerViewId的fragment，这也就会导致fragment生命周期方法的调用，但是remove方法并不会同步移除backstack中的记录，所以当我们点击返回按钮时仍然会返回fragment1，这也就是为什么只会调用onCreateView()和onViewCreated方法，却不会调用onCreate方法的原因，这样就导致我们每次返回就会重新走一遍onCreateView()和onViewCreated()内的逻辑，如果我们在这两个方法中进行了网络请求等操作，那么每次返回我们都会重新请求接口，这样很明显时不好的，下面有两种解决的方法

1，将网络请求入口放入到viewmodel中而不是fragment中，在页面销毁时通过onSaveInstance（)保存当前界面中view的状态，比如recyclerview的位置，比如checkbox的选择状态等等，然后onRestoreInstance(）中恢复相应的状态

2，在onCreateView()方法中保存当前界面view，在下次进入时判断当前界面view是否为null，如果为空的话就inflate布局，否者就返回保存的view

3，自己在activity中通过fragmentmanager来的hide，show，popBackStack方法管理fragment，在fragment中通过viewmodel+livedata通知activity进行页面的跳转逻辑
