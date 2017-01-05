说到EventBus大家一定不陌生,毕竟已经出来好几年而且github很多开元项目都使用了,但是目前很多帖子都还是讲的EventBus2.4,所以抽空学习整理了下Eventbus3.0的用法,本文主要讲解EventBus3.0的用法.

**什么是EventBus**
-------
EventBus是一个Android端优化的publish/subscribe消息总线，简化了应用程序内各组件间、组件与后台线程间的通信。比如请求网络，等网络返回时通过Handler或Broadcast通知UI，接口回调数据，这些需求都可以通过EventBus实现。
采用消息发布/订阅的一个很大的优点就是代码的简洁性，并且能够有效地降低消息发布者和订阅者之间的耦合度。

**1.EventBus初体验**
------

```
package com.zly.www.eventbusdemo.activity;

import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;

import com.zly.www.eventbusdemo.R;

import org.greenrobot.eventbus.EventBus;
import org.greenrobot.eventbus.Subscribe;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        EventBus.getDefault().register(this);//注册eventbus
        EventBus.getDefault().post("你好世界");//发送事件
    }

    @Subscribe
    public void receiverMessage(String msg){//接受事件方法
        Log.i("test", msg);
    }


    @Override
    protected void onDestroy() {
        super.onDestroy();
        EventBus.getDefault().unregister(this);//注销eventbus
    }


}
```
打印结果

![](http://img.blog.csdn.net/20160313152205905)

这就是一个Eventbus最简单的用法,只是在activity中添加了eventbus的注册,注销,发送事件,接收事件四步分别对应如下

注册eventbus
```
EventBus.getDefault().register(this);
```
注销eventbus
```
EventBus.getDefault().unregister(this);
```
发送事件
```
EventBus.getDefault().post("你好世界");
```
接受事件
```
@Subscribe
    public void receiverMessage(String msg){
        Log.i("test", msg);
    }
```
这里要注意

1. 发送事件和接收事件是靠着发送事件参数类型和接收事件方法参数类型相对应的.即这里EventBus.getDefault().post("你好世界")发送的事件为String类型,接收事件方法public void receiverMessage(String msg)参数也为String类型,所以可以接收到事件.

2. 在3.0之前，EventBus还没有使用注解方式。消息处理的方法也只能限定于onEvent、onEventMainThread、onEventBackgroundThread和onEventAsync，分别代表四种线程模型。而在EventBus3.0以后方法名你不必再去约定OnEvent为方法开头了,取而带之的是需要添加一个注解@Subscribe，其含义为订阅者,消息处理的方法可以随便取名，但要指定线程模型（默认为PostThread），四种线程模型，下面会讲到。注意，事件处理函数的访问权限必须为public，否则会报异常。

**2.线程模型**
-------
在EventBus的事件处理函数中需要指定线程模型，即指定事件处理函数运行所在的线程。EventBus中通常有四种线程模型，分别是PostThread（默认）、MainThread、BackgroundThread与Async。

PostThread：如果使用事件处理函数指定了线程模型为PostThread，那么该事件在哪个线程发布出来的，事件处理函数就会在这个线程中运行，也就是说发布事件和接收事件在同一个线程。

MainThread：如果使用事件处理函数指定了线程模型为MainThread，那么不论事件是在哪个线程中发布出来的，该事件处理函数都会在UI线程中执行。该方法可以用来更新UI，但是不能处理耗时操作。

BackgroundThread：如果使用事件处理函数指定了线程模型为BackgroundThread，那么如果事件是在UI线程中发布出来的，那么该事件处理函数就会在新的线程中运行，如果事件本来就是子线程中发布出来的，那么该事件处理函数直接在发布事件的线程中执行。在此事件处理函数中禁止进行UI更新操作。

Async：如果使用事件处理函数指定了线程模型为Async，那么无论事件在哪个线程发布，该事件处理函数都会在新建的子线程中执行。同样，此事件处理函数中禁止进行UI更新操作。
下面用例子来验证一下
一般情况下我们发送的事件都为对象类型而不是String所以这里改为发送对象

```
public class Event {

    public static class Message{
        public String msg;

        public Message(String msg) {
            this.msg = msg;
        }
    }

}
```

发布事件
```
Log.i("test", "post->" + Thread.currentThread().getId());
        EventBus.getDefault().post(new Event.Message("帅也是错吗"));//发送事件
```

接收事件
```
@Subscribe(threadMode = ThreadMode.POSTING)
    public void onEventPostThread(Event.Message msg) {
        Log.i("test", "onEventPostThread->" + Thread.currentThread().getId());
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onEventMainThread(Event.Message msg) {
        Log.i("test", "onEventMainThread->" + Thread.currentThread().getId());

    }

    @Subscribe(threadMode = ThreadMode.BACKGROUND)
    public void onEventBackgroundThread(Event.Message msg) {
        Log.i("test", "onEventBackgroundThread->" + Thread.currentThread().getId());
    }

    @Subscribe(threadMode = ThreadMode.ASYNC)
    public void onEventAsyncThread(Event.Message msg) {
        Log.i("test", "onEventAsyncThread->" + Thread.currentThread().getId());
    }
```

打印结果

![](http://img.blog.csdn.net/20160313160650284)

在将发送事件改为子线程发送,接收事件方法不变
```
new Thread(){
            @Override
            public void run() {
                super.run();
                Log.i("test", "post->" + Thread.currentThread().getId());
                EventBus.getDefault().post(new Event.Message("帅也是错吗"));//发送事件
            }
        }.start();
```
打印结果

![](http://img.blog.csdn.net/20160313160940136)

从测试可以看出上面结论准确无误

**3.事件接收优先级和取消事件**
---------
接收事件方法可以通过@Subscribe(priority = 1),priority的值来决定接收事件的顺序,数值越高优先级越大,默认优先级为0.(注意这里优先级设置只有在同一个线程模型才有效)

```
EventBus.getDefault().post(new Event.Message("帅也是错吗"));//发送事件
```

相同线程模型订阅事件
```
@Subscribe(threadMode = ThreadMode.POSTING, priority = 1)
    public void onEventPostThread1(Event.Message msg) {
        Log.i("test", "priority = 1");
    }

    @Subscribe(threadMode = ThreadMode.POSTING, priority = 2)
    public void onEventPostThread2(Event.Message msg) {
        Log.i("test", "priority = 2");
    }
```
打印结果

![](http://img.blog.csdn.net/20160313164757794)

相同线程模型下优先级高的先执行.

不同线程模型订阅事件
```
@Subscribe(threadMode = ThreadMode.POSTING, priority = 1)
    public void onEventPostThread1(Event.Message msg) {
        Log.i("test", "priority = 1");
    }

    @Subscribe(threadMode = ThreadMode.BACKGROUND, priority = 2)
    public void onEventPostThread2(Event.Message msg) {
        Log.i("test", "priority = 2");
    }
```
打印结果

![](http://img.blog.csdn.net/20160313164853081)

事件优先级不影响不同线程模型订阅事件顺序.

你可以取消事件订阅通过调用cancelEventDelivery(Object event)方法,任何进一步的事件交付将被取消,后续用户不会接收到事件,但是该方法只有在ThreadMode.PostThread事件处理方法中调用才有效.

发布事件
```
EventBus.getDefault().post(new Event.Message("帅也是错吗"));//发送事件
```

订阅事件
```
@Subscribe(threadMode = ThreadMode.MAIN, priority = 1)
    public void onEventPostThread1(Event.Message msg) {
        Log.i("test", "priority = 1");
    }

    @Subscribe(threadMode = ThreadMode.POSTING, priority = 2)
    public void onEventPostThread2(Event.Message msg) {
        Log.i("test", "priority = 2");
        EventBus.getDefault().cancelEventDelivery(msg);//取消仅限于在发布运行线程ThreadMode.PostThread事件处理方法。
    }
```
打印结果

![](http://img.blog.csdn.net/20160313170728604)

成功取消了事件

**4.粘性事件**
------
除了上面讲的普通事件外，EventBus还支持发送黏性事件。简单讲，就是在发送事件之后再订阅该事件也能收到该事件.

注册和注销方法一样,发送事件和订阅事件有些区别
发送事件
```
EventBus.getDefault().postSticky(new Event.Message("test"));
```
订阅粘性事件 默认sticky为false
```
@Subscribe(sticky = true)
    public void onEventPostThread(Event.Message msg) {
        .....
    }
```
接下来看一个完整的栗子
activity代码
```
package com.zly.www.eventbusdemo.activity;

import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;
import android.view.View;
import android.widget.Button;

import com.zly.www.eventbusdemo.R;
import com.zly.www.eventbusdemo.bean.Event;

import org.greenrobot.eventbus.EventBus;
import org.greenrobot.eventbus.Subscribe;
import org.greenrobot.eventbus.ThreadMode;

public class MainActivity extends AppCompatActivity {
    private int index = 0;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button bt_send = (Button) findViewById(R.id.bt_send);
        Button bt_regist = (Button) findViewById(R.id.bt_regist);
        Button bt_unregist = (Button) findViewById(R.id.bt_unregist);

        bt_send.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.i("test", "POSTING->" + index);
                EventBus.getDefault().postSticky(new Event.Message("test" + index++));
            }
        });
        bt_regist.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                EventBus.getDefault().register(MainActivity.this);
            }
        });
        bt_unregist.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                EventBus.getDefault().unregister(MainActivity.this);
            }
        });


    }


    @Subscribe(threadMode = ThreadMode.POSTING,sticky = true)
    public void onEventPostThread(Event.Message msg) {
        Log.i("test", "onEventPostThread->" + msg.msg);
    }

    @Subscribe(threadMode = ThreadMode.MAIN,sticky = true)
    public void onEventMainThread(Event.Message msg) {
        Log.i("test", "onEventMainThread->" + msg.msg);
    }

    @Subscribe(threadMode = ThreadMode.BACKGROUND,sticky = true)
    public void onEventBackgroundThread(Event.Message msg) {
        Log.i("test", "onEventBackgroundThread->" + msg.msg);
    }

    @Subscribe(threadMode = ThreadMode.ASYNC,sticky = true)
    public void onEventAsyncThread(Event.Message msg) {
        Log.i("test", "onEventAsyncThread->" + msg.msg);
    }

}

```
布局文件
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".activity.MainActivity">

    <Button
        android:id="@+id/bt_send"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="发送事件" />

    <Button
        android:id="@+id/bt_regist"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="注册" />

    <Button
        android:id="@+id/bt_unregist"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="注销" />

</LinearLayout>

```
代码很简单，界面上三个按钮，一个用来发送黏性事件，一个用来订阅事件，还有一个用来取消订阅的。首先在未订阅的情况下点击发送按钮发送一个黏性事件，然后点击订阅,打印如下:

![](http://img.blog.csdn.net/20160313180250252)

这就是粘性事件，能够收到订阅之前发送的消息。但是它只能收到最新的一次消息，比如说在未订阅之前已经发送了多条黏性消息了，然后再订阅只能收到最近的一条消息。这个我们可以验证一下，我们连续点击发送事件按钮发送黏性事件，然后再点击注册按钮订阅，打印结果如下：

![](http://img.blog.csdn.net/20160313180633865)

到这里EventBus3.0使用介绍完毕,还有什么可以补充的欢迎留言
