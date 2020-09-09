###AsyncTask源码分析

AsyncTask是我们经常用来执行异步任务时使用到的类，它让我们可以在子线程中执行一些操作，并且在UI线程上发送我们执行的结果，是我们更方便的进行子线程与主线程之间的交互，AsyncTask一般用来执行一些耗时较短的操作（一般可能就几秒钟），如果需要执行耗时时间较长的操作，还是建议使用ThreadPoolExecutor和FutureTask。

下面我们来看一下AsyncTask的使用

```java
 @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_asynctask);
		  //创建一个AsyncTask对象，注意AsyncTask是一个泛型类，需要传入三个泛型形参，这三个泛型形参对
		  //应的是doInBackground(）onProgressUpdate() onPostExecute()三个方法的参数类型
        AsyncTask<String, Integer, String> asyncTask = new AsyncTask<String, Integer, String>() {
       
       	  //用来处理任务的方法，在子线程中执行
            @Override
            protected String doInBackground(String... strings) {
                String name = Thread.currentThread().getName();
                Log.e("doInBackground", name);
                int i = 3;
                for (int j = 0; j < i; j++) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    publishProgress(10 * j);
                }
                return strings[0];
            }
			  //跟新进度，当在doInBackground(）中调用publishProgress()方法时会调用此方法，此方法在主线程中执行，常常用来跟新进度条等操作
            @Override
            protected void onProgressUpdate(Integer... values) {
                super.onProgressUpdate(values);
                String name = Thread.currentThread().getName();
                Log.e("onProgressUpdate", name);
                Log.e("onProgressUpdate", values[0] + "");
            }
				
			  //发送执行结果，参数即为执行结果，此方法也在主线程中执行
            @Override
            protected void onPostExecute(String s) {
                super.onPostExecute(s);
                String name = Thread.currentThread().getName();
                Log.e("onPostExecute", name);
            }
        };
        
        asyncTask.execute("test");
    }
```

执行上面的代码我们在三个方法中分别打印当前线程名称，来验证一下

```java
//doInBackground方法执行在AsyncTask #3线程中
03-29 10:55:03.816 29624-30482/com.kk.tongfu.threadpool E/doInBackground: AsyncTask #3
//onProgressUpdate执行在main线程中
03-29 10:55:04.856 29624-29624/com.kk.tongfu.threadpool E/onProgressUpdate: main
//每次调用publishProgress都会调用onProgressUpdate方法
03-29 10:55:04.857 29624-29624/com.kk.tongfu.threadpool E/onProgressUpdate: 0
03-29 10:55:05.897 29624-29624/com.kk.tongfu.threadpool E/onProgressUpdate: main
03-29 10:55:05.898 29624-29624/com.kk.tongfu.threadpool E/onProgressUpdate: 10
03-29 10:55:06.939 29624-29624/com.kk.tongfu.threadpool E/onProgressUpdate: main
03-29 10:55:06.940 29624-29624/com.kk.tongfu.threadpool E/onProgressUpdate: 20
//onPostExecute执行在主线程中
03-29 10:55:06.940 29624-29624/com.kk.tongfu.threadpool E/onPostExecute: main

```
从打印的log看出，三个方法只有doInBackground方法会在子线程中进行执行，其余两个方法都会在主线程中进行执行，可能有同学会觉得奇怪，一个对象中的不同方法为什么会执行在不同的线程呢，下面我们通过源码来一探究竟

在查看源码前，我们先来看几个跟AsyncTask息息相关的类

* ThreadPoolExecutor

> 线程池，用来创建线程执行任务的，当我们有大量的异步任务需要执行时，如果每个异步任务都开启一个线程执行，会大量增加性能开销，而线程池主要解决的就是此类问题，它减少了每个任务的调用开销，从而提升了在执行大量异步任务时的性能，线程池中的线程分为核心线程和非核心线程，当任务数量较少时，只使用核心线程来执行任务，当任务数量较多，我们会将其存储到任务队列中，当存储任务的队列已满，此时我们开启工作线程来执行任务，在工作线程执行完成后且消息队列中没有了消息时，工作线程就会被销毁，而核心线程依然保留，这样就避免了每次有任务来临时都要重新创建线程，较少了性能开销

* InternalHandler 

>一个Handler类的子类，用来进行主线程和子线程之间进行通信

* FutureTask

> 一个可以取消的异步任务，一般我们的任务都是Runnable和Callable对象，但是这两个对象只有run()和call()两个方法，无法提供取消任务，查询任务是否完成等功能，所以FutureTask在实现Runnable的基础上同时实现了Future接口，而Future接口定义了取消任务，查询任务状态，获取任务执行结果等操作，FutureTask对其进行了实现。

* WorkerRunnable<Params, Result>

>一个实现了Callable对象对象的抽象类，Callable<Result>类只记录了结果的泛型类型，而WorkerRunnable<Params, Result>在它的基础上有记录了参数的泛型类型

* ArrayDeque<E> 

> 一个基于数组实现的双端队列，内部是一个可以自增长的数组，可以从队列的头部和尾部取元素，也可以在队列的头部和尾部添加元素，当元素存满整个数组时，会新建一个容量更大的数组，并将原来的数组中的内容存入进去

了解了上面几个类的大致作用后，我们来开始分析AsyncTask的源码，首先我们从AsyncTask的构造函数看起

```java
 //无参数构造函数，一般我们使用此构造函数
 public AsyncTask() {
        this((Looper) null);
    }
 //传递一个handler给构造函数   
 public AsyncTask(@Nullable Handler handler) {
    //获取handler的looper对象作为参数传递给第三个构造函数，没有handler对象则返回null
    this(handler != null ? handler.getLooper() : null);
}
 //最终调用的构造函数，传递一个Looper对象来作为参数
 public AsyncTask(@Nullable Looper callbackLooper) {
        //判断Looper对象是否为null,如果为null,获取主线程的handler对象，如果不为null，但是looper对象是主线程的looper对象，我们也获取主线程
        //的handler对象，如果即不为null有不等于主线程的looper，那我们就使用当前looper来创建一个handler对象
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
        //创建一个WorkerRunnable对象，我们可以看到在WorkerRunnable对象的call()方法中调用了doInBackground(mParams)方法，这点我们后面再分析
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    //设置当前线程优先级
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    //调用doInBackground，并获取返回值
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    //发送执行结果
                    postResult(result);
                }
                return result;
            }
        };
		 //根据创建的WorkerRunnable对象来创建一个FutureTask对象
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```
可以看到，共有三个构造函数，最终会调用参数为Looper对象的构造函数，一般我们使用无参数构造函数（通过实验发现有参数的构造函数无法使用，不知道什么原因），使用无参数构造函数时，会通过getMainHandler()方法来创建一个持有主线程Looper的InternalHandler对象，那么通过此handler发送的消息会在主线程中进行执行，这个过程其实和我们在主线程创建handler，在子线程通过此handler发送消息最终会在主线程中执行是一样的过程，对Handler不了解的同学，可以看看我的这篇文章[Handler源码分析]()

在创建AsyncTask对象后，如果我们需要执行任务后返回的结果，我们会通过execute(Params... params)来开始执行任务，如果不需要返回的结果，我们会通过execute(Runnable runnable)方法来执行任务，我们先来看execute(Params... params)方法

```java
//注意此方法，在api<11时，此方法是在一个单线程中执行，api>=11,此方法会在一个线程池中进行执行
 public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```
方法中调用了executeOnExecutor方法

```java
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
            //判断当前状态，如果当前任务是在运行，或者运行完了，就抛出异常
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

		 //标记当前状态为运行中
        mStatus = Status.RUNNING;
        //执行前的回调方法，先于doInBackground方法Worker
        onPreExecute();
        
        //将传递的参数保存在构造函数中创建的WorkerRunnable对象中
        mWorker.mParams = params;
        //执行这个任务
        exec.execute(mFuture);

        return this;
    }
```
我们可以看到，在executeOnExecutor（）方法中我们先进行了一些状态的判断，然后标记当前任务的状态，最后调用exec.execute(mFuture)方法来执行任务，
exec对象就是上个方法中我们传递sDefaultExecutor，我们来看看它是一个什么东西

```java
 //下面两行代码可以看出sDefaultExecutor是一个volatile修饰的SerialExecutor常量，被volatile修饰的对象，在不同线程获取
 //它时获取到的都是最新的状态，这样就避免了出现线程安全问题
 public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
 private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
 //SerialExecutor 顾名思义串形执行器，它是用来存入和取出，以及执行任务的
 private static class SerialExecutor implements Executor {
        //定义一个双端队列
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        //记录当前正在运行的任务
        Runnable mActive;
       
        //开始执行
        public synchronized void execute(final Runnable r) {
            //首先将任务存入到队列中
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        //执行FutureTask的run方法
                        r.run();
                    } finally {
                        //执行下一个任务
                        scheduleNext();
                    }
                }
            });
            //如果还没有任务执行过，取一个任务来执行
            if (mActive == null) {
                scheduleNext();
            }
        }

		  //取任务执行
        protected synchronized void scheduleNext() {
            //从队列中取出任务
            if ((mActive = mTasks.poll()) != null) {
                //交给线程池来执行
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
可以看到，在调用exec.execute(mFuture)方法时，首先通过Runnable类将我们传入的FutureTask对象进行包装，保证我们在执行完FutureTask对象的run方法后通过调用scheduleNext()方法来取一个新的任务来执行，然后将我们包装完成的任务放到了队列中，然后判断是否有任务执行过，如果还没有任务执行过，从队列中取一个任务出来执行，任务的执行是通过线程池来进行的即THREAD_POOL_EXECUTOR.execute(mActive)方法，我们看看线程池THREAD_POOL_EXECUTOR是何时创建的

```java
//计算cpu的核心数
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
//计算核心线程数，在2-4之间
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
//计算最大线程数  是核心线程数的2倍加1
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
//计算工作线程最大空闲时间
private static final int KEEP_ALIVE_SECONDS = 30;

//创建一个线程工厂常量
 private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
//创建一个任务队列常量
private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);

//创建一个线程池常量
public static final Executor THREAD_POOL_EXECUTOR;

static {
    //通过静态代码块来创建线程池对象
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    //设置核心线程也有最大存活时间
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}

```
从上面的代码我们发现，首先我们通过cpu的核心数计算了一个核心线程数和最大线程数，然后通过这些内容来创建了一个核心线程有超时机制的线程池，然后通过线程池来执行我们取出的任务
至于线程池是如何工作的，这里就不再展开讲了，具体可以看我的另一篇文章[线程池的工作机制](),在上一步的分析中我们通过Runnable类将我们传入的FutureTask对象进行包装，那么线程池在执行任务的时候创建一个线程并执行其run方法，调用的也就是上一步Runnable对象的run方法，在其内部调用的是FutureTask的run方法，而FutureTask对象在我们创建AsyncTask时就已创建，在创建FutureTask对象时我们传入了一个WorkerRunnable(一个Callback类的实现类),执行FutureTask的run方法其实就是执行WorkerRunnable对象的call方法，我们再来回顾一下WorkerRunnable对象的call方法做了什么操作

```java
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    //设置线程的优先级
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    //调用doInBackground() 方法，此处可以看出，此方法是在线程池创建的子线程中调用的
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    //发送消息
                    postResult(result);
                }
                return result;
            }
        };
```
WorkerRunnable对象的call方法中我们调用了doInBackground(mParams)方法，这也就解释了为什么doInBackground(mParams)方法会在子线程中调用，可以看到在finally模块中，我们调用了 postResult(result)来将执行结果发送了出去

```java
 private Result postResult(Result result) {
        //使用我们创建的持有主线程looper的handler对象来创建一个message对象，并将doInBackground()方法执行的结果封装成AsyncTaskResult对象传递给了message对象
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
        new AsyncTaskResult<Result>(this, result));
        //通过sendToTarget将消息发送给handler对象
        message.sendToTarget();
        return result;
    }
```
上面的方法将执行结果封装成了一个AsyncTaskResult对象，并传递给了message对象，然后发送给了handler对象来执行，我们来看看handler对象做了什么

```java
 private static class InternalHandler extends Handler {
       
       //传递主线程的Looper对象给handler对象，这样handlermessage方法就会在主线程进行执行
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            //取出消息中的AsyncTaskResult对象
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            //判断消息类型
            switch (msg.what) {
                //如果是发送结果
                case MESSAGE_POST_RESULT:
                    //调用当前AsyncTask对象的finish方法
                    result.mTask.finish(result.mData[0]);
                    break;
                //如果是发送进度
                case MESSAGE_POST_PROGRESS:
                    //调用当前AsyncTask对象的onProgressUpdate()方法
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
   
private void finish(Result result) {
    //判断是否取消了任务执行
    if (isCancelled()) {
        //如果取消了执行此方法
        onCancelled(result);
    } else {
        //如果没有取消，执行此方法
        onPostExecute(result);
    }
    //将当前AsyncTask对象的状态标记为finish
    mStatus = Status.FINISHED;
} 
    
```
可以看到我们根据消息类型来判断是调用finish方法还是onProgressUpdate()方法，finish()方法内部判断当前任务是否被取消了，如果没有被取消执行onPostExecute(result)方法，什么时候到执行到onProgressUpdate()呢？我们来看一个publishProgress方法

```java
 protected final void publishProgress(Progress... values) {
        //如果没有取消
        if (!isCancelled()) {
            //使用我们创建的持有主线程looper的handler对象来创建一个message对象，并将doInBackground()方法执行的结果封装成AsyncTaskResult对象传递给了message对象
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
            new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```
可以看到，当我们调用publishProgress方法时，会发送一个消息，而这个消息就是更新进度的消息，这样onProgressUpdate()方法就会执行到

至此AsyncTask工作流程差不多已经分析完毕。
在开始的分析中我们说过，如果我们需要执行任务后返回的结果，我们会通过execute(Params... params)来开始执行任务，如果不需要返回的结果，我们会通过execute(Runnable runnable)方法来执行任务，下面我们来看看execute(Runnable runnable)的执行过程

```java
public static void execute(Runnable runnable) {
        sDefaultExecutor.execute(runnable);
    }
```
你会发现，它并没有将runnable对象进行包装成FutureTask对象，而是直接将其交给了sDefaultExecutor来执行，而sDefaultExecutor又将其交给了线程池来执行，这样它在执行玩自己的run方法后就完成了，而run方法是没有返回值的，所以当我们不关心返回值，不需要更新进度等，我们可以直接使用这种方式来进行异步任务的执行。

说了那么多，看起来很复杂，其实可以总结为以下步骤
> * 1  AsyncTask类内部维护了一下线程工厂，一个内部维护了一个双端队列的任务执行器SerialExecutor对象，一个持有主线程Looper的handler对象(相当于在主线程创建了一个handler对象)
> * 2  执行任务时先将任务添加进双端队列，然后从队列中取出一个任务，交给线程工厂来执行
> * 3  线程工厂创建线程来执行这个任务，在执行完成后，通过handler发送一个消息到主线程，并在主线程中处理了这个消息，根据不同的消息类型，执行不同的方法

好了，到此基本分析完毕，如有不足的地方，欢迎指正