#OKHttp源码分析

okHttp是安卓使用非常流行的网络请求框架，它提供了连接池来实现连接的复用，来进一步减少资源的消耗，同时提供了失败重连等功能，它提供了拦截器的功能，能够让我们在进行网络请求的时候来修改请求信息和封装返回信息，下面我们的简单分析一下它的源码

###基本使用

####OkHttpClient的创建

 ```java
 OkHttpClient mClient;
 OkHttpClient.Builder builder = new OkHttpClient.Builder();
 mClient = builder
                .connectTimeout(5, TimeUnit.SECONDS)
                .readTimeout(5, TimeUnit.SECONDS)
                .build();
 ```
上面可以看到，我们通过构造者模式创建了一个OkHttpClient对象，然后为它设置了连接超时和读取超时，一般来说，一个应用只需要创建一个OkHttpClient对象就够了，然后在网络请求的时候使用这同一个对象即可,因为每个OkHttpClient对象都有它自己的连接池和线程池，OkHttpClient正是因为使用了连接池和线程池，通过重用链接和线程来减少延迟，节省内存，如果每个网络请求都创建一个OkHttpClient对象，势必会造成资源浪费，OkHttp的优势也就不复存在

####get请求
```java
       //get请求
 		 Request.Builder requestBuilder = new Request.Builder();
        Request request = requestBuilder.url("https://www.baidu.com")
                .get()
                .build();
		//创建一个call对象
        Call call = mClient.newCall(request);
       //异步进行网络请求
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //网络请求失败
                Log.e("onFailure",e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                //网络请求成功
                //response  返回的数据
                Log.e("onResponse",response.body().string());
                Log.e("onResponse",response.toString());
            }
        });
```

####post请求

```java
        //创建一个request对象
        Request.Builder builder=new Request.Builder();
        //创建一个jsonobject对象，存储数据
        JSONObject jsonObject=new JSONObject();
        try {
            jsonObject.put("phone","13020163733");
            jsonObject.put("code","9999");
            jsonObject.put("password","123456");
        } catch (JSONException e) {
            e.printStackTrace();
        }
		 //创建一个requestbody对象
        RequestBody body1 = RequestBody.create(MediaType.parse("application/json"), jsonObject.toString());
		
       Request request = builder.url("http://www.baidu.com")
        .post(body1)
        .build();
        
        //创建一个call对象
        Call call = mClient.newCall(request);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //网络请求失败
                Log.e("onFailure",e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                //网络请求成功
                Log.e("onResponse",response.body().string());
                Log.e("onResponse",response.toString());
            }
        });

```

在上面的get和post方法示范中，我们可以看到，OkHttpClient和Request都是使用了构建者模式来创建对象，我们都是先创建了一个Request对象，然后通过OkHttpClient的newCall(request)方法创建一个Call对象，然后调用call对象的.enqueue方法进行异步请求，不同的是，post方法创建了一个ReqeustBody对象，作为一个参数传递给了request对象，下面我们来一步步分析Okhttp网络请求的过程

###创建OkHttpClient对象

OkHttpClient对象使用构造者模式来进行创建，我们来看看Builder的构造方法做了什么

```java
Dispatcher dispatcher;
    @Nullable Proxy proxy;
    List<Protocol> protocols;
    List<ConnectionSpec> connectionSpecs;
    final List<Interceptor> interceptors = new ArrayList<>();
    final List<Interceptor> networkInterceptors = new ArrayList<>();
    EventListener.Factory eventListenerFactory;
    ProxySelector proxySelector;
    CookieJar cookieJar;
    @Nullable Cache cache;
    @Nullable InternalCache internalCache;
    SocketFactory socketFactory;
    @Nullable SSLSocketFactory sslSocketFactory;
    @Nullable CertificateChainCleaner certificateChainCleaner;
    HostnameVerifier hostnameVerifier;
    CertificatePinner certificatePinner;
    Authenticator proxyAuthenticator;
    Authenticator authenticator;
    ConnectionPool connectionPool;
    Dns dns;
    boolean followSslRedirects;
    boolean followRedirects;
    boolean retryOnConnectionFailure;
    int callTimeout;
    int connectTimeout;
    int readTimeout;
    int writeTimeout;
    int pingInterval;

    public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      if (proxySelector == null) {
        proxySelector = new NullProxySelector();
      }
      cookieJar = CookieJar.NO_COOKIES;
      //Socket创建工厂
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      //链接池
      connectionPool = new ConnectionPool();
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      callTimeout = 0;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
    }
```
可以看到在我们创建OkHttpClient.Builder()对象时，其实是为OkHttpClient做了一些包括创建Socket工厂和链接池以内的一些初始化操作，同时Builder类对象也为我们提供了一些方法来更改其中的值，比如下面两个我们常用的设置连接超时的方法

```java

    @IgnoreJRERequirement
    public Builder connectTimeout(Duration duration) {
      connectTimeout = checkDuration("timeout", duration.toMillis(), TimeUnit.MILLISECONDS);
      return this;
    }

   
    public Builder readTimeout(long timeout, TimeUnit unit) {
      readTimeout = checkDuration("timeout", timeout, unit);
      return this;
    }
    
```
最后通过Builder对象的build方法讲Builder对象中的值传递给了OkHttpClient,至此OkHttp对象创建完成

###创建Request对象

从文章的开头我们可以看到，Request对象同样是通过构造者模式来创建的，来看看Request.Builder构造函数做了什么操作

```java
@Nullable HttpUrl url;
String method;
Headers.Builder headers;
@Nullable RequestBody body;

/** A mutable map of tags, or an immutable empty map if we don't have any. */
Map<Class<?>, Object> tags = Collections.emptyMap();

public Builder() {
  //调用无参数构造函数，默认是get方法
  this.method = "GET";
  //创建一个Heanders对象
  this.headers = new Headers.Builder();
}

 //设置请求的url
 public Builder url(String url) {
  if (url == null) throw new NullPointerException("url == null");
  //根据url判断是http请求还是https请求
   if (url.regionMatches(true, 0, "ws:", 0, 3)) {
    url = "http:" + url.substring(3);
  } else if (url.regionMatches(true, 0, "wss:", 0, 4)) {
    url = "https:" + url.substring(4);
  }
  return url(HttpUrl.get(url));
}
    
//设置请求方式为get
public Builder get() {
  return method("GET", null);
}
    
//设置请求方式为post
public Builder post(RequestBody body) {
  return method("POST", body);
}
//判断请求的方法和请求RequestBody是否为null，赋值给全局变量
 public Builder method(String method, @Nullable RequestBody body) {
  if (method == null) throw new NullPointerException("method == null");
  if (method.length() == 0) throw new IllegalArgumentException("method.length() == 0");
  if (body != null && !HttpMethod.permitsRequestBody(method)) {
    throw new IllegalArgumentException("method " + method + " must not have a request body.");
  }
  if (body == null && HttpMethod.requiresRequestBody(method)) {
    throw new IllegalArgumentException("method " + method + " must have a request body.");
  }
  this.method = method;
  this.body = body;
  return this;
}   
```
从上面的几个方法中可以看到，Request.Builder类为我们提供了一些设置请求方法，设置请求路径，设置post请求参数等方法，通过调用这些方法，我们可以得到一个Request对象。

###创建RealCall对象
在示例中，我们使用mClient.newCall(request)方法创建一个Call对象

```java
 //通过Client的newCall方法创建一个Call对象
 Call call = mClient.newCall(request);
```
看看OkHttpClient内部做了什么操作

```java
	@Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```
在OkHttpClient内部通过调用调用RealCall.newRealCal(this, request, false）方法来创建一个Call对象，接着看RealCall类

```java
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
```
可以看到，我们在newRealCall方法中创建了一个RealCall对象并返回，同时创建了一个EventListener对象用来进行网络请求的过程
的回调，自此Call对象创建完毕

###RealCall.enqueue(Callback responseCallback)方法开始进行网络请求

在上面的示例中，我们通过一下调用来开始进行异步网络请求
```java

		 //异步进行网络请求
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //网络请求失败
                Log.e("onFailure",e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                //网络请求成功
                //response  返回的数据
                Log.e("onResponse",response.body().string());
                Log.e("onResponse",response.toString());
            }
        });
        
我们知道call对象是RealCall类的对象，我们来看看RealCall.enqueue方法做了什么操作

```java
@Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      //判断当前call对象是否被执行过
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    //调用callstart方法说明请求已经开始
    eventListener.callStart(this);
    //使用Dispatcher对象来执行这个任务
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
在上面的代码中我们创建了一个new AsyncCall(responseCallback)对象，AsyncCall类是Runnable类的子类，看看run方法执行了什么操作

```java
 @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      //调用了当前对象的execute方法
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }
```

再来看AsyncCall的execute方法

```java
 @Override protected void execute() {
      boolean signalledCallback = false;
      timeout.enter();
      try {
        //获取接口返回结果
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          //判断网络请求是否被取消
          signalledCallback = true;
          //如果取消调用responseCallback.onFailure方法，
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          //如果没有取消
          signalledCallback = true;
          //调用responseCallback.onResponse(RealCall.this, response)方法
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        e = timeoutExit(e);
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
```
可以看到，在execute()方法中我们获取到了网络请求结果，并进行结果的回调，既然AsyncCall是一个Runnable对象，并且是通过下面的一句代码来执行的

```java
client.dispatcher().enqueue(new AsyncCall(responseCallback));
```
我们来找到OkHttpClient对象的dispatcher()方法

```java
public Dispatcher dispatcher() {
    return dispatcher;
  }
```

这个dispatcher常量是在哪赋值的呢

```java
 public Builder() {
      dispatcher = new Dispatcher();
      ...省略无关代码...
    }
```
可以看到，dispatcher是我们在创建OkHttpClient对象时创建的，我们来看看它的enqueue方法

```java
private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

 void enqueue(AsyncCall call) {
    synchronized (this) {
      readyAsyncCalls.add(call);
    }
    promoteAndExecute();
  }
```
异步任务被添加到了双端队列中，然后调用了promoteAndExecute()方法

```java
private boolean promoteAndExecute() {
    //以当前dispatcher为锁
    assert (!Thread.holdsLock(this));
	 //用来保存可执行的Call对象
    List<AsyncCall> executableCalls = new ArrayList<>();
    //是否运行
    boolean isRunning;
    synchronized (this) {
      //循环从双端队列中取出异步任务
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();
		  //如果正在运行的异步请求任务多余最大的请求数，跳出循环，不在向里面添加
        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        //如果正在运行的异步请求任务与当前任务的host相同的数量大于单个host的最大请求数，跳出循环，不在向里面添加
        if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.
		  //从双端队列中移除当前异步请求任务
        i.remove();
        //向可执行列表中添加当前异步任务
        executableCalls.add(asyncCall);
        //向正在运行的列表中添加当前异步任务
        runningAsyncCalls.add(asyncCall);
      }
      //如果正在运行的任务的数量大于0，标记当前任务正在执行
      isRunning = runningCallsCount() > 0;
    }

	// 循环取出可执行网络请求任务
    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      //在executorService()的返回值中执行网络请求任务
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }
```
来看看executorService()方法的返回值

```java
public synchronized ExecutorService executorService() {
    if (executorService == null) {
      //创建一个核心线程数量为0，超时销毁线程时间为60s的线程池
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```
可以看到，我们将AsyncCall异步任务放在了线程池中来执行，来看看AsyncCall对象的executeOn(）方法

```java
 void executeOn(ExecutorService executorService) {
      assert (!Thread.holdsLock(client.dispatcher()));
      boolean success = false;
      try {
        executorService.execute(this);
        success = true;
      } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        eventListener.callFailed(RealCall.this, ioException);
        responseCallback.onFailure(RealCall.this, ioException);
      } finally {
        if (!success) {
          client.dispatcher().finished(this); // This call is no longer running!
        }
      }
    }
```

可以看到线程池执行了execute方法，即执行了这个可执行任务，也就是调用了AsyncCall对象的run方法，即execute()方法，由于是在线程池中执行，所以此方法是在子线程中执行，我们再一次来看看execute()方法

```java
Override protected void execute() {
      boolean signalledCallback = false;
      //开始进入连接计时，连接超时的逻辑即在timeout对象中，
      timeout.enter();
      try {
        //通过getResponseWithInterceptorChain()来获取网络请求方法的返回值
        Response response = getResponseWithInterceptorChain();
        ... 省略无关代码 ...
            }
   
```
可以看出，在上面的方法中通过getResponseWithInterceptorChain()方法来获取当前网络请求结果，也就是说网络请求其实是在getResponseWithInterceptorChain()方法中进行的，来看看getResponseWithInterceptorChain()方法

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    //首先获取我们设置的拦截器
    interceptors.addAll(client.interceptors());
    //添加关键的五个拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    //根据拦截器生成一个拦截器链
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());
    //使用拦截器链处理当前请求
    return chain.proceed(originalRequest);
  }
```

// 可以看到我们通过设置拦截器来生成一个拦截器链，然后通过拦截器来处理请求，看看chain.proceed(originalRequest)方法

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    ...省略次要代码...
    //调用下一个拦截器
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    //通过拦截器的intercept方法来执行
    Response response = interceptor.intercept(next);
    ...省略次要代码...
    return response;
  }
```
在proceed方法中，我们通过获取拦截链中的拦截器，然后通过拦截器的intercept方法来层层解析请求并进行网络请求，然后获取返回值，再层层封装后作为返回值返回给最初的拦截器，如下图所示

<img src="https://raw.githubusercontent.com/tongfuzz/image/master/interceptchain.png" />

从上面我们可以看到，这几个拦截器才是网络请求的关键所在，下面我们来逐个分析拦截器的处理过程

###RetryAndFollowUpInterceptor



