###Handler源码分析
   
   Handler是我们经常使用的一个类，我们可以用它来进行线程间的通信，也可以用它来执行延时任务等。当我们创建一个新的Handler对象时，会把它和创建它的线程以及线程中的消息队列进行绑定，这样它就可以传递消息和任务给线程中的消息队列，并且在消息队列取出这个消息时执行它。下面我们来看一下Handler的两种基本的使用方式：

* 1 在主线程中使用

```java
   @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Handler handler=new Handler(){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                //重写handleMessage方法，接收消息
                if(msg.what==1){
                    Log.e("Handler","接收到消息了");
                }
            }
        };
			
		 //创建一个消息对象
        Message message = handler.obtainMessage();
        message.what=1;
        handler.sendMessage(message);
    }

```

* 2 在子线程中使用

```java
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

		 //创建一个新的线程
        new Thread(){
            @Override
            public void run() {
            	   //调用Looper.prepare()来创建一个消息队列供handler使用
                Looper.prepare();
                //获取looper对象传递给handler
                final Handler handler=new Handler(Looper.myLooper()){
                    @Override
                    public void handleMessage(Message msg) {
                        Log.e("handleMessage","getMessage");
                    }
                };
                //发送两个空延时消息
                handler.sendEmptyMessageDelayed(10,1000);
                handler.sendEmptyMessageDelayed(20,2000);
                //开始循环取出消息队列中消息来交给handler执行
                Looper.loop();//注意，此处是一个阻塞的方法，她下面的方法内容一般执行不到，除非调用了quite方法
            }
        }.start();
    }
``` 

通过上面的两种方式我们可以看出，在主线程中创建并不需要调用Looper.prepare()和Looper.loop()，这主要是因为在应用创建的时候就已经创建了looper对象并且调用了loop方法来开始取消息执行。
下面我们开始来分析源码，首先我们来看handler的构造函数

```java
 public Handler() {
 		  //调用了两个参数的构造方法
        this(null, false);
    }
 //handler的无参构造函数会调用此构造函数   
 public Handler(Callback callback, boolean async) {
 		  
 		  //判断handler在创建时是否被static关键字修饰，如果没有，可能会导致内存泄漏
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
		
		 //通过调用Looper.myLooper()来获取一个looper对象，赋值给mLooper变量
        mLooper = Looper.myLooper();
        //判断mLooper是否为null，如果为null抛出异常，
        if (mLooper == null) {
        	  //当前线程没有调用Looper.prepare()，所以无法创建一个handler对象，由此可以看出，looper对象在调用Looper.prepare()时候创建，在调用
        	  //Looper.myLooper()时获取创建的对象
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        //取出Looper中的消息队列对象赋值给mQueue变量
        mQueue = mLooper.mQueue;
      	 //用来处理消息的回调，可能为null
        mCallback = callback;
        //消息的执行是否是异步的
        mAsynchronous = async;
    }
```
在构造函数中，我们通过Looper.myLooper()来取Looper对象，如果取到为空，就会抛出异常，根据异常消息提示我们可以看到，它需要我们先调用Looper.prepare(), 才能在线程中创建handler对象，来看看Looper.prepare()做了什么

```java

	public static void prepare() {
        prepare(true);
    }
	
    private static void prepare(boolean quitAllowed) {
    	 //从ThreadLocal对象中取出looper对象，如果取到了就说明这个线程已经有了looper对象，不能再创建一个，所以抛出异常，这就说明一个线程只能有一个looper
    	 //对象，如果在一个线程中多次调用Looper.prepare()就会抛出异常
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        //如果取不到，就重新创建一个Looper对象，并保存到ThreadLocal对象中
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
在prepare方法中我们首先判断当前线程中是否已经有了一个looper对象，如果有了就抛出异常，如果没有就新建一个looper对象并保存到ThreadLocal对象中，可能有些同学会觉得疑惑它是如何保证一个线程中只有一个looper对象呢，其实这是ThreadLocal的功劳，因为ThreadLocal在每个线程中对该变量会创建一个副本，即每个线程内部都会有一个该变量，且在线程内部任何地方都可以使用，线程之间互不影响，这样一来就不存在线程安全问题，也不会严重影响程序执行性能，这也就保证了每个线程都有自己的looper对象，其实它的内部就是通过在线程类Thread创建一个Map集合，将ThreadLocal对象和对应的值作为键值对保存到了Map中，具体不再深入叙述，有时间会写一篇关于ThreadLocal的文章，感兴趣的同学也可以自己去查看一下源码

在上面的代码中，我们通过new Looper(quitAllowed)创建了一个looper对象，我们来看看Looper对象的构造函数

```java
 private Looper(boolean quitAllowed) {
 		 //创建一个消息队列
        mQueue = new MessageQueue(quitAllowed);
        //获取当前线程
        mThread = Thread.currentThread();
    }
```
在构造函数中我们创建了一个消息队列，并获取了当前线程，这样Looper对象就持有一个消息队列和一个线程，而handler对象则持有这个looper对象，这样消息队列，线程，handler我们都有了，接下来就开始发送消息了

我们来看handler如何发送消息，通常我们会通过Message.obtain()来创建一个message对象并通过handler对象的sendMessage(Message msg)来发送消息，来看看sendMessage做了什么操作

```java
 public final boolean sendMessage(Message msg)
    {	  //调用了sendMessageDelayed方法,延时时间为0
        return sendMessageDelayed(msg, 0);
    }
             
  public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
    	  //判断延时时间是否小于0，如果小于0则延时时间置为0
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        //调用sendMessageAtTime来发送消息，此方法在指定时间来发送消息，延时消息其实就是在当前时间加上延时时间这一时刻来发送消息
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
   public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
   		 //判断消息队列是否为null
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        //通过调用enqueueMessage来处理消息，参数为消息队列，消息，和执行时间
        return enqueueMessage(queue, msg, uptimeMillis);
    }

```
从上面的代码可以看出sendMessage(Message msg)最终调用了enqueueMessage(queue, msg, uptimeMillis）来处理这个消息，下面我们来看看此方法做了什么操作

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
		 //设置message对象的指定对象，即为当前handler对象
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        //调用MessageQueue对象的enqueueMessage(msg, uptimeMillis)方法来处理消息
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
可以看到，我们将消息最终交给了消息队列MessageQueue对象来处理，注意msg.target = this这句代码，它让message对象持有了当前handler，将两者进行了绑定，保证每个消息都有执行它的handler对象，下面我们接着看MessageQueue的enqueueMessage(msg, uptimeMillis)方法

```java
 boolean enqueueMessage(Message msg, long when) {
 		  //判断消息的目标是否为null，在前面我们已经将handler对象作为target
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        //判断消息是否在使用
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }
		 //线程安全
        synchronized (this) {
        	  //如果退出了消息派发
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                //消息回收，返回false，消息发送不成功
                msg.recycle();
                return false;
            }
			  //标记消息在使用中
            msg.markInUse();
            //标记消息何时执行
            msg.when = when;
            //mMessages记录消息链
            Message p = mMessages;
            //是否需要唤醒消息队列，如果消息队列是阻塞的，当来新的消息时需要唤醒消息队列
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                //说明此消息是头消息，即第一个消息
                
                //msg.next 指向下一个消息，此处指向null
                msg.next = p;
                //将此消息作为头消息
                mMessages = msg;
                //如果阻塞需要唤醒消息队列
                needWake = mBlocked;
            } else {
                //如果在阻塞，并且头消息没有target指向，并且此消息需要异步执行，那么就唤醒消息队列，开始消息派发
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                //从消息链表头部开始循环
                Message prev;
                for (;;) {
                    prev = p;
                    //取出当前消息的下一个消息
                    p = p.next;
                    //如果下一个消息为null，则跳出循环，在链表末尾执行插入消息操作。如果下一个消息不为null，但是要添加的消息的执行时间小于下
                    //一个消息的执行时间(即先于下一个消息执行)，此时跳出循环，将要添加的消息插入到下一个消息的前面
                    if (p == null || when < p.when) {
                    
                        break;
                    }
             
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                //跳出循环后开始插入消息
                msg.next = p;
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                //如果需要唤醒消息队列，唤醒队列
                nativeWake(mPtr);
            }
        }
        return true;
    }

```
可以看到，在enqueueMessage方法中我们生成了一个消息列表，在链表的头部是执行时间点最小的消息，在链表尾部是执行时间点最大的消息，这样我们就能从链表头部开始取消息，依次按时执行，在这个方法中，我们只是生成了消息链表，那么在什么时候开始取消息执行呢，在文章的开头用法示例，我们调用了Looper.prepare()后紧接着创建了handler对象，然后发送了消息，最后我们调用了 Looper.loop()方法，既然发送消息只是生成了消息链表，那么Looper.loop()应该就是(其实就是)开始取消息执行的地方了，我们来看看Looper.loop()做了什么

```java
public static void loop() {
        //获取Looper对象
        final Looper me = myLooper();
        //判断是否调用了Looper.prepare()，即是否创建了looper对象
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //获取looper对象的消息队列
        final MessageQueue queue = me.mQueue;

        // 没看明白
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            //取出下一个消息，可能会阻塞
            Message msg = queue.next(); // might block
            //如果取不到，就返回
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
			  
			  //这部分代码是log的打印，直接跳过
            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            final long start = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            final long end;
            try {
                //调用了msg.target的dispatchMessage(msg)方法，也就是handler的dispatchMessage(msg)
                msg.target.dispatchMessage(msg);
                end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (slowDispatchThresholdMs > 0) {
                final long time = end - start;
                if (time > slowDispatchThresholdMs) {
                    Slog.w(TAG, "Dispatch took " + time + "ms on "
                            + Thread.currentThread().getName() + ", h=" +
                            msg.target + " cb=" + msg.callback + " msg=" + msg.what);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
			  //回收消息
            msg.recycleUnchecked();
        }
    }
```
从上面的代码中可以看出，通过当前线程的MessageQueue对象的next()方法来获取消息，他是如何获取的呢，如何和我们上面生成的消息链表产生关联的呢，我们来看看MessageQueue的next()方法做了什么操作

```java
Message next() {

		  //将Native层的messagequeue引用地址返回给Java层保存在mPtr变量中，通过这种方式将Java层的对象与Native层的对象关联在了一起
        final long ptr = mPtr;
        //如果Native层的messagequeue引用地址为0就返回null
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        //下次取消息的时间
        int nextPollTimeoutMillis = 0;
        for (;;) {
            
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            //这是一个native方法，实际作用就是通过Native层的MessageQueue阻塞nextPollTimeoutMillis毫秒的时间
            nativePollOnce(ptr, nextPollTimeoutMillis);
			
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                //如果一个消息没有设置target，他的作用是插入一个消息屏障，在这个屏障之后的所有的额同步消息都不会执行
                if (msg != null && msg.target == null) {
                    // 循环执行，知道找到一个异步执行的消息，由于插入了消息屏障，所以所有的同步消息都不会执行，所以我们找异步消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                //如果消息不为null
                if (msg != null) {
                    //判断当前时间是否小于消息执行时间点
                    if (now < msg.when) {
                        //如果小于消息执行时间点，说明还没有到时间执行此消息，记录两者之间的时间差，作为下次取消息的时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        //如果不小于于消息执行时间点，说明已经到了当前消息的执行时间点，不再阻塞
                        mBlocked = false;
                        //取出消息，并将此消息的下一个消息赋值给mMessages
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        //标记消息在使用中
                        msg.markInUse();
                        /返回消息
                        return msg;
                    }
                } else {
                    // 说明没有更多的消息了，将下次取消息的时间置为-1
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    //如果消息队列退出了，返回false
                    dispose();
                    return null;
                }

					//下面是关于一些IdleHandler的一些判断和执行，一般使用不到，就不再分析了
              
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
从上面的代码我们知道，首次进入循环nextPollTimeoutMillis=0，阻塞方法nativePollOnce(ptr, nextPollTimeoutMillis)会立即返回，然后读取列表中的消息，如果发现消息屏障，则跳过后面的同步消息，如果没有符合条件的消息，会处理一些不紧急的任务（IdleHandler），如果取出了消息就返回此消息，获取消息的方法和添加消息的方法都涉及到了native层的代码，感兴趣的同学可以研究下

上面我们说到，取到消息后我们会通过msg.target.dispatchMessage(msg)来进行消息的派发，即将消息给msg所持有的handler的dispatchMessage(msg)方法来执行

```java
 public void dispatchMessage(Message msg) {
 		  //如果消息设置了callback，交给消息的callback来执行，一般通过post(Runnable r)方法发送runnable对象是会将
 		  //runnable对象赋值给msg.callback，会走到此处
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            //如果消息没有设置callback，则交给handler的callback来执行，
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            //如果都没有设置，则交给handler的handleMessage方法来执行，即我们经常重写的方法
            handleMessage(msg);
        }
    }
```
至此，handler消息传递基本就完成了

在我们平常的使用中，除了通过sendMessage(Message message)方法来发送消息，我们也可以通过post(Runnable r)方法来发送消息，其实本质是一样的，只不过post方法中传递的runnable对象被赋值给了Message对象的callback变量，这样我们就会走到dispatchMessage(Message msg)的第一个if方法中。

在文章的开头我们提到过，主线程创建Handler时之所以不用传递Looper对象，是因为主线程已经创建了Looper对象，我们知道一个线程只能有一个Looper对象，如果多次调用 Looper.prepare(),程序会抛出异常，下面我们根据这点来验证一下，我们创建一个Activity，在构造onCreate函数中调用Looper.prepare()

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Looper.prepare();


    }

}
```
运行程序你会发现直接崩溃，并报出如下错误信息
```java
 java.lang.RuntimeException: Unable to start activity ComponentInfo{com.kk.tongfu.handler/com.kk.tongfu.handler.MainActivity}:                     
 java.lang.RuntimeException: Only one Looper may be created per thread
```
说明主线程中确实创建了Looper对象。


至此，handler消息传递机制分析完成，描述可能有些乱，下次加油！！