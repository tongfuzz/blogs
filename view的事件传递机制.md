###view的事件传递机制的分析

在activity启动过程中我们分析到，在performResumeActivity()中我们通过创建ViewRootImpl对象，并通过setView方法将我们创建的DecorView与其进行绑定，ViewRootImpl是视图管理对象，用来处理windowmanager和view之间的交互
在setView的过程中，我们会创建一个InputChannel对象(它也是一个Binder对象)用来发送事件到另一个进程中的window对象，同时我们也会创建一个WindowInputEventReceiver对象
```java
 ViewRootImpl.java
 public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    	 ...忽略部分代码
    	 if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    //创建InputChannel对象，用来从WMS到Window对象间发送事件
                    mInputChannel = new InputChannel();
                }
    	 ...忽略部分代码
        if (mInputChannel != null) {
            if (mInputQueueCallback != null) {
                //创建用来存储InputEvent的队列
                mInputQueue = new InputQueue();
                mInputQueueCallback.onInputQueueCreated(mInputQueue);
            }
            //创建一个WindowInputEventReceiver,绑定InputChannel和looper对象，内部使用原生方法
            mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                    Looper.myLooper());
        }
        ...忽略部分代码
    }
}
    }
```
在WindowInputEventReceiver中，我们会通过调用onInputEvent()方法来进行处理输入事件

```java
 ViewRootImpl.java
 final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            super(inputChannel, looper);
        }

        @Override
        public void onInputEvent(InputEvent event, int displayId) {
            //处理InputEvent事件
            enqueueInputEvent(event, this, 0, true);
        }

        @Override
        public void onBatchedInputEventPending() {
            if (mUnbufferedInputDispatch) {
                super.onBatchedInputEventPending();
            } else {
                scheduleConsumeBatchedInput();
            }
        }

        @Override
        public void dispose() {
            unscheduleConsumeBatchedInput();
            super.dispose();
        }
    }
```

```java
 ViewRootImpl.java
 void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        adjustInputEventForCompatibility(event);
        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);

        //从队列中取出最后一个InputEvent对象
        QueuedInputEvent last = mPendingInputEventTail;
        if (last == null) {
            mPendingInputEventHead = q;
            mPendingInputEventTail = q;
        } else {
            last.mNext = q;
            mPendingInputEventTail = q;
        }
        //处理事件个数加1
        mPendingInputEventCount += 1;
        Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                mPendingInputEventCount);
        //如果需要立即处理
        if (processImmediately) {
            doProcessInputEvents();
        } else {
            scheduleProcessInputEvents();
        }
    }

```

```java
ViewRootImpl.java
void doProcessInputEvents() {
        //循环取出InputEvent
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            if (mPendingInputEventHead == null) {
                mPendingInputEventTail = null;
            }
            q.mNext = null;

            mPendingInputEventCount -= 1;
            Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                    mPendingInputEventCount);

            long eventTime = q.mEvent.getEventTimeNano();
            long oldestEventTime = eventTime;
            if (q.mEvent instanceof MotionEvent) {
                MotionEvent me = (MotionEvent)q.mEvent;
                if (me.getHistorySize() > 0) {
                    oldestEventTime = me.getHistoricalEventTimeNano(0);
                }
            }
            mChoreographer.mFrameInfo.updateInputEventTime(eventTime, oldestEventTime);
			  //传递InputEvent事件
            deliverInputEvent(q);
        }

        if (mProcessInputEventsScheduled) {
            mProcessInputEventsScheduled = false;
            mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
        }
    }
```
调用deliverInputEvent(q)开始传递事件

```java
ViewRootImpl.java
private void deliverInputEvent(QueuedInputEvent q) {
       ...忽略次要代码
       InputStage stage;
        if (q.shouldSendToSynthesizer()) {
            stage = mSyntheticInputStage;
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
        }

        if (stage != null) {
            //传递事件到SyntheticInputStage()对象中
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    }
```
传递事件到InputState中
```java
ViewRootImpl.InputStage.java
public final void deliver(QueuedInputEvent q) {
            if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
                forward(q);
            } else if (shouldDropInputEvent(q)) {
                finish(q, false);
            } else {
                //调用onProcess(q)方法来处理事件
                apply(q, onProcess(q));
            }
        }
```

```java
final class EarlyPostImeInputStage extends InputStage {
       ...忽略无关代码
        @Override
        protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);
            } else {
                final int source = q.mEvent.getSource();
                //如果事件源是触摸屏幕或者是鼠标
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    //开始传递事件
                    return processPointerEvent(q);
                }
            }
            return FORWARD;
        }
       ...忽略无关代码
    }
```
上面传递事件的过程中通过mNext对象会先传递到ViewPostImeInputStage类中的processPointerEvent(QueuedInputEvent q)方法中
```java
 private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;
            mAttachInfo.mUnbufferedDispatchRequested = false;
            mAttachInfo.mHandlingPointerEvent = true;
            //调用了DecorView的dispatchPointerEvent(event)方法中
            boolean handled = mView.dispatchPointerEvent(event);
            maybeUpdatePointerIcon(event);
            maybeUpdateTooltip(event);
            mAttachInfo.mHandlingPointerEvent = false;
            if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
                mUnbufferedInputDispatch = true;
                if (mConsumeBatchedInputScheduled) {
                    scheduleConsumeBatchedInputImmediately();
                }
            }
            return handled ? FINISH_HANDLED : FORWARD;
        }
```
在此方法中会调用到DecorView中的dispatchPointerEvent(event)，也就是View类中的dispatchPointerEvent(event)方法

```java
View.java
public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            //此时就走到了DecorView的dispatchTouchEvent(event)方法中
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }
```

```java
DecorView.java
public boolean dispatchTouchEvent(MotionEvent ev) {
    //mWindow对像就是PhoneWindow对象
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```
可以看到我们调用了PhoneWindow的callback对象的dispatchTouchEvent(ev)方法，在我们调用performLaunchActivity()方法我们创建完Activity后会调用Activity的attach()方法

```java
 Activity.java
 final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        ...忽略无关代码
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        //可以看到我们PhoneWindow对象的Callback对象就是当前Activity对象
        mWindow.setCallback(this);
        ...忽略无关代码
    }
```
此时便走到了Activity的dispatchTouchEvent()方法中，下面就开始了事件在视图中的传递

  view的事件传递机制是安卓学习要掌握的重要知识点之一，正好最近在复习，写篇文章记录一下：
  我们知道当我们点击了屏幕就产生了触摸事件，这个事件被封装成了一个MotionEvent对象并且传递给了当前activity的dispatchTouchEvent方法，那么我们从activity类的dispatchTouchEvent开始入手
 
```java
  	 public boolean dispatchTouchEvent(MotionEvent ev) {
  	 	//首先判断是否是down事件
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        	//此方法让用户在产生点击事件时自己做一些事情，这个方法是一个空实现
        	//与事件传递机制关系不大
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```
    
从上面的代码可以看出，通过调用getWindow().superDispatchTouchEvent(ev)方法，activity将点击事件传递给了Window的superDispatchTouchEvent(ev)方法，下面首先来看getWindow方法

```java
public Window getWindow() {
    return mWindow;
}
```java    

然后我们来看mWindow在哪个地方被赋值

```java
 final void attach(Context context,...) {
   
   ...省略
   
    mWindow = new PhoneWindow(this);
    
   ...省略
}
```
我们发现mWindow其实是一个PhoneWindow对象，也就是说getWindow().superDispatchTouchEvent(ev)其实调用的是PhoneWindow的superDispatchTouchEvent(ev)方法，那么接下来我们进入PhoneWindow类中看一看superDispatchTouchEvent方法

```java
	@Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```
可以看到它内部调用的是mDecor.superDispatchTouchEvent(event)方法，我们来查看mDecor在什么地方被赋值

```java
	 public PhoneWindow(Context context, Window preservedWindow,
            ActivityConfigCallback activityConfigCallback) {
        ...省略
        if (preservedWindow != null) {
            mDecor = (DecorView) preservedWindow.getDecorView();
            ...省略
        }
    }
``` 
    
可以看到mDecor是通过一个window对象的getDecorView()方法返回的，由于window是一个抽象类而且getDecorView是一个抽象方法，而PhoneWindow是Window的唯一实现类所以我们直接看PhoneWindow类中的getDecorView方法

```java
	@Override
    public final View getDecorView() {
        if (mDecor == null || mForceDecorInstall) {
            installDecor();
        }
        return mDecor;
    }
```    
由于mDecor刚开始是一个空对象，我们会installDecor()方法，顾名思义这是一个创建DecorView的方法，下面我们来看此方法做了什么操作，此方法比较长，我做了精简，只看主相关代码

```java
 private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
    	  //创建DecorView对象
        mDecor = generateDecor(-1);
        ...省略
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
    }
}
```   
在此方法中我们通过调用generateDecor来创建了一个DecorView对象(其实就是一个FrameLayout对象，因为DecorView是FrameLayout的子类)然后我们通过generateLayout(mDecor)方法来给mDecor对象添加子view，

```java
	protected ViewGroup generateLayout(DecorView decor) {
        // 此方法主要用来给DecorView设置一些标志为，并且根据设置的主题加载不同的布局到mDecor中，此方法较长，只保留了核心代码
			...省略
        mDecor.startChanging();
        //加载布局文件到mDecor中
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
		//找到布局文件中id为com.android.internal.R.id.content的view，我们所写的xml布局文件就是被加载到了此view中
		...省略
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
                return contentParent;
    }
```    
在此方法中我们根据当前主题，加载相应的布局文件到DecorView对象中，并从布局文件中找到我们的内容布局，即我们平常写的xml加载进的布局，并复制给mContentParent对像，我们顺便来看一下我们常用的setContentView做了什么操作

```java
	public void setContentView(int layoutResID) {
		...省略
		//可以看到此方法通过infalte方法将我们的布局加载进了mContentParent对象中
       mLayoutInflater.inflate(layoutResID, mContentParent);
       ...省略
       }
```     
  
至此，我们的布局层级已经形成了DecorView(FrameLayout) ——> 根据不同主题加载的不同布局(LinearLayout) ——> id为R.id.content的布局(FrameLayout)  ——> 我们自己写的xml文件

我们回到PhoneWindow 的superDispatchTouchEvent方法中，activity的方法最终调用到此方法，而此方法又调用了
mDecor.superDispatchTouchEvent(event)

```java
	 @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```    
    
我们进入到DecorView的superDispatchTouchEvent(event)看一看

```java	
	public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```
由于DecorView是继承自FrameLayout，由于FrameLayout继承自ViewGroup并且并未重写dispatchTouchEvent方法，所以super.dispatchTouchEvent(event)最终调用的是ViewGroup的dispatchTouchEvent(event)方法，至此我们才进入到事件传递的主要逻辑，下面我们来分析此方法中具体做了什么操作，此方法代码较长，相关无用代码我会删除，我们只关注主流程(看源代码不能太深入细节，不然我们会越看越晕，只关注主要流程即可)

```java
	@Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        
        //如果这个事件指向的是能获取焦点的view并且当前view能获取焦点
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
        	
        		//执行正常的事件派发过程
            ev.setTargetAccessibilityFocus(false);
        }
		
		//事件的处理结果
        boolean handled = false;
        //当此事件应该被分发时返回true，应该被舍弃时放回false
        if (onFilterTouchEventForSecurity(ev)) {
        		//获取当前action
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;
            
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                //如果是down事件表明这是一个新的点击事件，清除以前点击事件的一些状态和标志位，在此时，我们清					
                //除了touchtarget，使得mFirstTouchTarget为null；
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            //判断是否拦截
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                //如果是down事件，或者mFirstTouchTarget不为null
                /*FLAG_DISALLOW_INTERCEPT不允许接受事件的标志,ViewGroup提供了一个
                * requestDisallowInterceptTouchEvent来改变这个标记，默认是允许的，即
                * ~FLAG_DISALLOW_INTERCEPT
                * 所以这个里的disallowIntercept默认是false
                */
               final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                		//调用onInterceptTouchEvent方法来判断是否拦截当前事件，默认返回false
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                		//如果不允许接受事件的标志，intercepted为false，说明不拦截
                    intercepted = false;
                }
            } else {
                /**
                执行到此处说明当前action不是down事件，并且mFirstTouchTarget为null，说明没有子view来消费 					这个事件，于是拦截这个事件交给自己处理
               	*/
                intercepted = true;
            }
            
            if (intercepted || mFirstTouchTarget != null) {
            		//如果拦截，或者mFirstTouchTarget不为null，执行正常的事件分发
                ev.setTargetAccessibilityFocus(false);
            }

            //判断事件是否取消
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            //是否需要分发给多个view，比如多个view的叠加
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            
            //保存当前要分发的对像
            TouchTarget newTouchTarget = null;
            //是否已经分发给当前对象的标记
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {
				
                //如果这个事件分发给了能够获取焦点的view但是它并没有处理它，那么我们将它分发给他的子view
                
                //由于刚刚我们已经设置为false，所以childWithAccessibilityFocus为null
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    // 如果是down事件，多点触控事件
                    
                    //如果是down事件，actionIndex返回值为0，此方法用来获取触摸事件的序号 以及触摸点的id信息。
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    //将触摸目标链表中与触摸点id相同的TouchTarget移除
                    removePointersFromTouchTargets(idBitsToAssign);
						
						//子view的数量
                    final int childrenCount = mChildrenCount;           
                    if (newTouchTarget == null && childrenCount != 0) {
                    //如果要分发的对像为null并且子view的数量不为0
                    
                    	//获取当前点击事件的xy坐标
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        //customOrder 默认为false
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;

                        for (int i = childrenCount - 1; i >= 0; i--) {
                        	//遍历子view
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            //在子view中找到这个能获取焦点的view
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }
                            //如果这个view不能能获取到点击事件，或者点击点不在这个view的范围内
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }
                            //执行至此已经找到能获取焦点并且点击点在这个view范围内的view


                            //获取此view的touchtarget，如果没有返回null
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                            		
                                //这个子view已经接收touch事件了，那么将此事件id作为新的事件id赋值给此子view，此时点击事件已经与此子view进行了绑定，跳出循环									
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }
								  //
								
                            resetCancelNextUpFlag(child);
                            //将事件向子view传递
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            		//如果dispatchTransformedTouchEvent返回true，说明此view消费了这个事件        
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                //添加一个新的touchtarget，并将其赋值给mFirstTouchTarget
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                //说明已经有view消费了这个事件
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            //说明没有子view消费这个事件，继续执行正常的派发流程
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // 没有找到一个子view来消费这个事件
                        // 将此事件交给最近添加的目标view来处理来处理         
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }
			
            // 将事件分发分发给相应的view.
            if (mFirstTouchTarget == null) {
                // 如果没有view处理这个事件，那么将这个事件交给当前view来处理
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // 将事件分发给相应的view来进行处理
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    //如果是down事件，mFirstTouchTarge==newTouchTarget
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        //一般down事件会走到此处，因为只有down事件才会循环遍历子view去寻找处理事件的view
                        handled = true;
                    } else {
                        //一般不是down事件都会走到这里
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        //将事件分发给目标view        
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }
```
在上面的传递过程中，我们会使用mFristTouchTarget来记录当前view中接收到此事件的view，如果是down事件，我们会通过遍历子view，调用子view的dispatchTouchEvent方法来判断是否处理此事件，如果子view是viewgroup，同样会执行上面的操作，如果所有的view遍历完成以后找到了一个view来处理这个事件，他的dispatchTouchEvent方法返回true作为dispatchTransformTouchEvent()的值，这样我们就会通过addTouchTarget方法来给mFristTouchTarget赋值，同时alreadyDispatchedToNewTouchTarget置为true,如果没有找到我们就会调用自己的onTouchEvent来处理事件，如果自己的onTouchEvent放回false，那么交给上层view的onTouchEvent处理，这样层层调用，如果所有的onTouchEvent放回false，那就说明没有view来处理事件，那么事件就交给了DecorView的onTouchEvent来处理

view事件传递到子view是通过dispatchTransformTouchEvent方法来进行的，我们来看看这个方法做了什么

```java

 private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        //如果是cancel事件，我们直接传递给子view
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
			
		  ...略去无关代码
        
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                //如果要传递的子view为null
                if (child == null) {
                    //调用父类的dispatchTouchEvent(event)方法，也就是view的						//dispatchTouchEvent(event)方法
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);
						//传递给自己的dispatchTouchEvent(event)方法
                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // Perform any necessary transformations and dispatch.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }
```

可以看到，事件传递过程中，如果子view为null ，就交给父类的dispatchTouchEvent方法处理，如果不为null，交给自身的dispatchTouchEvent方法处理了，viewgroup的父类为view类，看看view类的dispatchTouchEvent

```java
public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }

        boolean result = false;

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }

        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            //如果设置了OnTouchListener并且onTouch(this, event)方法返回true，那么次方法返回true
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            //如果没有设置onTouchListener或者设置了但是onTouch方法返回false,判断onTouchEvent方法，
            //如果onTouchEvent方法返回true，那么此方法返回true，注意，在onTouchEvent方法中我们会在
            //action_up状态下判断是否有设置onClickListener ,如果设置了，调用onClickListener的
            //onClick方法
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }

        // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }
```
