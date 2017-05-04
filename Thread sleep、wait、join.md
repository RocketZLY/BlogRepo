# Thread sleep、wait、join使用

这里先介绍join,然后把两个有关联的sleep和wait一起介绍.

## join()

这个方法比较好理解,当前线程等待指定线程终止后在执行,将两个交替执行的线程合并为顺序执行的线程.比如在B线程中调用A线程的join()方法,直到A线程执行完毕,B线程才会继续执行.

api有两个

- void join()

  当前线程等待调用这个方法的线程终止后再执行.

- void join(long millis)

  当前线程等待millis毫秒后,不管调用这个方法的线程是否终止,当前线程将变成可运行状态

这里我们不妨看一下join的源码,join()中调用的join(long millis)所以这里我只贴出join(long millis)方法

![](http://of1ktyksz.bkt.clouddn.com/thread_join%28%29.png)

可以发现他先调用isAlive()方法判断该线程是否存活,然后通过Object.wait(0)方法使当前线程阻塞(客官别急wait后面会讲到).

![](http://of1ktyksz.bkt.clouddn.com/thread_isAlive%28%29.png)

通过注释可以发现线程存活指的是线程存活并且调用start()了.接下来我们通过一个例子体会下join的使用


```java
public class MyClass {

    public static void main(String[] args) {

        System.out.println("main start");

        Thread t = new Thread(new MyRunnable());
        t.start();

        try {
            t.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("main end");

    }

    private static class MyRunnable implements Runnable {

        @Override
        public void run() {
            System.out.println("other start");

            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("other end");
        }
    }

}
```

运行结果.

![](http://of1ktyksz.bkt.clouddn.com/thread_join%28%29_result1.png)

接下来我们将join()方法替换成join(1000)

运行结果.

![](http://of1ktyksz.bkt.clouddn.com/thread_join%28%29_result2.png)

这里应该没啥问题就不多说了.

## sleep()

Thread静态方法,强制当前线程休眠.当休眠时间到期后恢复到可运行状态.不一定会立即执行,具体取决于线程调度器.

下面一个很简单的栗子

```java
public static void main(String[] args) {

    System.out.println("main start");

    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    System.out.println("main end");

}
```

结果就是先打印main start,然后休眠3秒后打印,main end.

我主要想说的是下面这个问题,**sleep()在Synchronized块中调用,线程虽然休眠了,但是对象的锁并未释放**,其他线程无法访问这个对象.下面一个例子展示这个过程.

```java
public static void main(String[] args) {

        MyRunnable r = new MyRunnable();
        Thread t = new Thread(r);
        t.start();
        r.method1();
    }

    private static class MyRunnable implements Runnable {

        @Override
        public synchronized void run() {
            System.out.println("other start");

            System.out.println("other end");
        }

        public synchronized void  method1(){
            System.out.println("main start");

            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("main end");
        }
    }
```

结果如下

![](http://of1ktyksz.bkt.clouddn.com/thread_sleep%28%29_synchronized.gif)

可以发现即使method1()中调用了Thread.sleep(),但对象锁并未释放,所以休眠3秒后依次执行main end->other start->other end.这里可以留意下后面在wait()方法的时候我们会做个对比.

## wait()

wait()一般都是配合notify()和notifyAll()一起使用,并且他们都是java.lang.Object的方法.

具体细节如下

- wait()方法是使当前线程,进入到一个和对象相关的等待池中,同时失去对象的锁,其他线程可以访问.

- wait()方法通过notify(),notifyAll(),或者指定超时时间来唤醒当前等待池中线程.

- wait()必须用在synchronized代码块中,否则会抛出异常.

下面通过一个栗子检验下

```java
public static void main(String[] args) {

    MyRunnable r = new MyRunnable();
    Thread t = new Thread(r);
    t.start();
    r.method1();
}

private static class MyRunnable implements Runnable {

    @Override
    public synchronized void run() {
        System.out.println("other start");

        System.out.println("other end");

        notifyAll();
    }

    public synchronized void method1() {
        System.out.println("main start");

        try {
            wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("main end");
    }
}
```

运行结果如下

![](http://of1ktyksz.bkt.clouddn.com/thread_wait_result.png)

由于线程创建需要时间,所以先执行了method1()方法打印出main start,然后调用wait()方法后该线程进入等待状态,并且释放该对象锁,于是run()运行打印出other start和other end,然后调用notifyAll()后wait的线程被唤醒打印出main end.  

这里细说下notify()与notifyAll(),

- notify()

  举个例子如果线程A1,A2,A3都obj.wait(),则B调用obj.notify()只能唤醒A1,A2,A3中的一个(具体哪一个由JVM决定)  

- notifyAll()

  obj.notifyAll()则能全部唤醒A1,A2,A3,但是要继续执行obj.wait()的下一条语句,必须获得obj锁,因此,A1,A2,A3只有一个有机会获得锁继续执行,例如A1,其余的需要等待A1释放obj锁之后才能继续执行.

## sleep()与wait()区别

- sleep()

  1. sleep是Thread静态方法,因此他不能改变对象的机锁,所以当在一个Synchronized块中调用Sleep()方法是,线程虽然休眠了,但是对象的锁并木有被释放,其他线程无法访问这个对象(即使睡着也持有对象锁).


- wait()

  1. wait()是Object类里的方法,当一个线程执行到wait()方法时,它就进入到一个和该对象相关的等待池中,同时失去了对象的锁,其他线程可以访问.

  2. wait()使用notify或者notifyAlll或者指定睡眠时间来唤醒当前等待池中的线程

  3. wait()必须放在synchronized block中，否则会在运行时抛出"java.lang.IllegalMonitorStateException"异常.

一句话总结最大区别:sleep()睡眠时,保持对象锁,仍然占有该锁;而wait()睡眠时,释放对象锁.

## 总结

1. join将并行运行的线程改为串行.

2. sleep使当前线程休眠,目的是不让当前线程独自霸占该进程所获的CPU资源,以留一定时间给其他线程执行的机会.

3. 在某种条件下该线程wait(),当条件成熟后通过notify()唤醒该线程继续运行.
