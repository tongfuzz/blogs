### RxJava 源码分析

rxjava是一个功能强大的库，我们常常用它来做一些线程转换的操作，下面我们来根据使用方式来分析一下它的源码

```java
Observable<String> ob = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                Log.e("subscribe",Thread.currentThread().getName()+"");
                emitter.onNext("haha");
                emitter.onComplete();
                Log.e("subscribe","complete");
                //emitter.onError(new Throwable("出错了"));
            }
        });

        Disposable subscribe = ob.
                subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        Log.e("accept", Thread.currentThread().getName() + "");
                        Log.e("accept", s + "");
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(Throwable throwable) throws Exception {
                        Log.e("accept", throwable.getMessage() + "");
                    }
                }, new Action() {
                    @Override
                    public void run() throws Exception {
                        Log.e("run", "run");
                    }
                }, new Consumer<Disposable>() {
                    @Override
                    public void accept(Disposable disposable) throws Exception {
                        Log.e("accept", "onSubscribe");
                    }
                });
```

首先看Observable.create方法，在这个方法中我们创建了一个ObservableOnSubscribe