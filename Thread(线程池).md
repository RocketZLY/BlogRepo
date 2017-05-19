# Thread->线程池

## 前言Callable与Future

在介绍线程池前,我们先介绍下Callable与Future因为等会封装异步任务会用到.而异步任务Runnable相信都在熟悉不过了,Callable与Runnable类似,但Callable有返回值.

```java
public interface Callable<V> {

    V call() throws Exception;
}
```

类型参数就是返回值类型,比如`Callable<Integer>`表示一个返回值类型为Integer的Callable.

至于异步任务Callable的执行结果会通过一个Future对象返回,Future接口具体有如下方法.

```java
public interface Future<V> {
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
}
```

- get()方法获取Callable执行结果,如果还未执行完将会阻塞直到可以获得结果,get(long timeout, TimeUnit unit)调用超时会抛出TimeoutException异常,如果运行该计算的线程被中断了,两个方法都将抛出InterruptedException.如果执行已经完成,那么get方法立即返回.

- 如果callable还在执行,isDone()方法返回false,如果完成了返回true.

- 可以用cancel取消计算,如果计算还没开始,它将被取消不在开始,如果计算已经运行中,那么mayInterruptIfRunning为true,它就被中断.

可以说Future里面不仅带着执行结果,并且如果你想中断当前计算也可以通过它.

使用的时候我们需要用FutureTask包装一下,FutureTask可以将Callable转换成Future和Runnable,它同时实现了二者的接口,下面我们来看一下如何使用

```java
public class MyClass {

    public static void main(String[] args){

        FutureTask<Integer> task = new FutureTask<>(new MyCallable());
        new Thread(task).start();
        try {
            task.get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }

    static class MyCallable implements Callable<Integer> {

        @Override
        public Integer call() throws Exception {
            return 100;
        }
    }
}
```

使用非常简单这里就不在过多赘言.

## Executors线程池

创建一个新的线程是有一定代价的,如果程序创建了大量生命周期很短的线程,就应该使用线程池.

执行器Executors有很多静态工厂方法可以构建线程池,具体如下

|方法|描述|
|---|---|
|newCachedThreadPool|必要时创建新线程,空闲线程会被保留60秒|
|newFixedThreadPool|创建一个定长线程池,空闲线程会被一直保留,可控制线程最大并发数,超出的线程会在队列中等待|
|newSingleThreadExecutor|创建一个单线程化的线程池,它只会用唯一的工作线程来执行任务,保证所有任务按照指定顺序执行|
|newScheduledThreadPool|创建一个定长线程池,支持定时及周期性任务执行.代替Timer|
|newSingleThreadScheduledeExecutor|用于预定执行的单线程池|

### 线程池

这里我们分两类介绍这些线程池,先看前三个.

- newCachedThreadPool 方法构造的线程池,对于每个任务,如果有空闲线程可用,立即让它执行,如果没有空闲线程则创建新的线程.

- newFixedThreadPool 方法构造一个具有固定大小的线程池,如果提交的任务数多于空闲的线程数,那么把得不到服务的任务放置在队列中.

- newSingleThreadExecutor 是一个大小为1的线程池,由一个线程执行提交的任务.

这三个返回实现了ExecutorService接口的ThreadPoolExecutor类的对象.

我们可以通过ExecutorService接口的submit与execute方法执行异步任务.

```java
void execute(Runnable command);

Future<?> submit(Runnable task);
<T> Future<T> submit(Runnable task, T result);
<T> Future<T> submit(Callable<T> task);
```

execute比较简单,这里我们说下submit方法.

- 第一个submit方法返回一个`Future<?>`,可以使用这个对象调用isDone,cancel,或者isCancel.但是get方法在完成的时候只是返回null

- 第二个submit方法在执行完成后返回一个指定的result对象.

- 第三个submit提交一个Callable,并且返回对应的Future.

顺带说下它们的区别

1. 接收的参数不同

2. submit有返回值,而execute没有

3. submit方便Exception处理,而execute处理异常比较麻烦.

当用完一个线程池的时候,调用shutdown,会使该线程池关闭,被关闭的线程池不在接受新的任务.当所有任务执行完了,线程池关闭.另一种是调用shutdownNow,该线程池取消还未开始的所有任务并试图中断还在运行的线程.

说了这么多来个例子看看

```java
public static void main(String[] args){

    ExecutorService threadPool = Executors.newCachedThreadPool();//创建一个线程池
    Future<Integer> future = threadPool.submit(new MyCallable(1));//提交一个任务
    try {
        System.out.println(future.get());//获得执行结果
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
    threadPool.shutdown();//关闭线程池
}

static class MyCallable implements Callable<Integer> {
    private int i;

    public MyCallable(int i) {
        this.i = i;
    }

    @Override
    public Integer call() throws Exception {
        Thread.sleep(i * 1000);
        return i;
    }
}
```

下面我们总结下使用线程池该做的

1. 调用Executors类中的静态方法获取线程池

2. 调用submit提交Runnable或Callable

3. 如果想取消一个任务,或者想得到执行的结果那么需要保存好Future对象

4. 当不在提交任何任务的时候,调用shutdown关闭线程池.

### 预定任务

接下来看另外两个线程池,Executors的newScheduledThreadPool和newSingleThreadScheduledeExecutor两个方法返回的实现了ScheduledExecutorService接口的对象,ScheduledExecutorService接口具有预定执行和重复执行的方法,可以预定执行Runnable或者Callable在初始延迟后运行一次,也可以周期性的执行一个Runnable.

ScheduledExecutorService常用方法如下

- `public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);`

  `public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);`

  在预定时间后执行任务

- `public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit);`

  在初始化延迟结束后,周期性的执行任务,周期执行间隔period.不受任务执行时间影响.

- `public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit);`

  在初始化延迟结束后,周期性的执行给定任务,受任务执行时间影响,当第一次执行完后延迟delay后再执行第二次.

下面一个简单的例子,每隔两秒执行一次的定时任务.

```java
ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();

scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override public void run() {
        //do something
    }
}, 0, 2000, TimeUnit.MILLISECONDS);
```

### 执行一组任务

前面我们已经介绍了如何使用线程池执行单个任务了,但有的时候需要执行一组任务,控制相关任务,这里我们介绍另外两组api.

```java
<T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;
<T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;

<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException;
```

invokeAny方法提交一个Callable对象集合到线程池,并返回某个已经完成了的任务结果,无法知道究竟是哪个任务的结果.适用于比如要计算一个结果但是有很多种方式,但只要有一种方式计算出结果我们就不在需要计算了这种情况.

invokeAll方法提交一个Callable对象集合到线程池,返回一个Future对象列表,代表所有任务的结果.

到这里线程池基础就介绍完了,欢迎吹水.
