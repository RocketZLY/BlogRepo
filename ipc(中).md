# IPC(中)
# 1  Android中IPC方式

在第一篇[IPC(上)](http://blog.csdn.net/zly921112/article/details/54089500)中我们已经介绍了IPC的基础知识:序列化和Binder,本篇将详细介绍各种跨进程通讯方式.具体有如下几种:

- Intent中extras传递

- 共享文件

- Binder

- ContentProvider

- Socket

# 1.1  Bundle

四大组件中的三大组件(Activity,Service,Receiver)都是支持在Intent中传递Bundle数据的,由于Bundle实现了Parcelable接口,所以他可以方便在不同进程间传输,**所以在我们开启另一个进程的Activity,Service,Receiver时候,就可以使用Bundle的方式来**,但是有一点需要注意,我们在Bundle中的数据必须可以被序列化,比如基本数据类型,实现了Parcelable接口的对象,实现了Serializable接口的对象等等,具体支持类型如下

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29_bundle.png)

如果是Bundle不支持的类型我们无法通过它在进程间通讯.但有的时候可以适当改变下实现方式来解决问题,比如A进程进行计算,得到结果后给到B进程,但是结果的数据类型Bundle不支持传递,那么这个时候我们可以将计算过程放在B进程的后台服务中,然后当需要计算的时候A进程通过Intent告知B进程的Service开始计算了,由于Service在B进程所以可以很方便的拿到数据,这样就成功避免了进程间通讯的问题.

# 1.2  使用文件共享

共享文件也是一种进程间通讯的方式,**两个进程通过读/写同一个文件来交换数据**,交换信息除了文本信息外,还可以序列化对象到文件在从另一个进程中读取这个对象,但是有一点需要注意,Android基于Linux,并发读/写文件没有限制,当两个线程同时写文件的时候可能会出现问题,这里尤其需要注意.下面是序列化对象到文件共享数据的栗子

这次我们在MainActivity中,序列化一个User对象到文件,在另一个进程运行的SecondActivity中反序列化,看对象的属性值是否相同.

```java
// MainActivity
public void serialize(View v) {
        User user = new User("zhuliyuan", 22);
        try {

            File file = new File(getCacheDir(), "user.txt");
            FileOutputStream fos = new FileOutputStream(file);
            ObjectOutputStream oos = new ObjectOutputStream(fos);
            oos.writeObject(user);

            fos.close();
            oos.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

//SecondActivity
public void deserialize(View v) {
        File file = new File(getCacheDir(), "user.txt");
        if (file.exists()) {
            try {
                FileInputStream fis = new FileInputStream(file);
                ObjectInputStream ois = new ObjectInputStream(fis);
                User user = (User) ois.readObject();

                fis.close();
                ois.close();

                Log.i("yyjun", user.toString());

            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

```

下面看下日志

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29%E6%96%87%E4%BB%B6%E5%85%B1%E4%BA%AB.png)

可以发现SecondActivity成功恢复了User数据,这里虽然数据相同但是和之前MainActivity的User对象并不是同一个.

通过文本共享这种方式共享数据对文本格式没有要求,只要双方按约定好格式即可.但是也有局限性当并发读/写的时候读取的文件可能不是最新的,如果并发写就更加的严重了,要尽量避免这种情况或者使用线程同步来限制并发.通过上面分析我们可以知道,**文件共享方式适合在对数据同步要求不高的进程间进行通信,并且需要妥善处理并发问题.**

当然SharedPreferences是个特例,它通过键值对方式存储数据,在底层上采用xml文件来存储键值对,一般情况每个应用的SharedPreferences文件目录位于/data/data/package name/shared_prefs目录下.从本质上来说SharedPreferences也是属于文件的一种,但是由于系统对它的读写有一定的缓存策略,所以内存中会有一份SharedPreferences缓存,而在多进程模式下,系统对他读写变得不可靠,当高并发时候有很大几率丢失数据,因为,在多进程通讯的时候最好不要使用SharedPreferences.

api文档中也明确指出了这一点

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29SharePreferences.png)

# 1.3  使用Messenger

Messenger可以翻译为信使,通过它可以在不同进程中传递Message对象,在Message中放 入我们想传递的数据,就可以实现数据的进程间传递了.Messenger底层实现是AIDL,这个可以通过构造方法初见端倪

```java
public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }

public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
```

是不是可以明显看出AIDL的痕迹,既然这么像那我们就通过源码来分析下,车总是要发的不过且慢,我觉得咱们先看下Messenger的栗子热热身,在来分析更好

实现步骤如下

1. 服务端进程

  首先在服务端创建一个Service来处理客户端的连接请求,同时创建一个Handle并通过它来创建一个Messenger对象,然后在Service中的onBind中返回这个Messenger对象底层的Binder即可

2. 客户端进程

  客户端进程中,首先需要绑定服务端service,绑定成功后用服务端返回的IBinder对象创建一个Messenger,通过这个Messenger向服务端发送消息,发送的消息类型为Message对象,如果需要服务端能够回应客户端,就必须和服务端一样,创建一个Handle并创建一个新的Messenger,并把这个Messenger对象通过Message的replyTo参数传递给服务端,服务端就可以通过replyTo参数回应客户端.

下面先来个简单的栗子,此栗中服务端无法回应客户端

在service中创建一个handle,然后new一个Messenger将Handle作为参数传入,再在onBind方法中返回Messenger底层的Binder

```java
public class MessengerService extends Service {

    private static final String TAG = "MessengerService";

    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case Constants.TYPE_MSG_FROM_CLIENT:
                    Log.i(TAG, "receiver client msg " + msg.getData().getString("msg"));
                    break;
            }
        }
    }

    private Messenger mMessenger = new Messenger(new MessengerHandler());

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
```

在注册service让其在单独的进程

```
<service android:name=".MessengerService"
            android:process=":remote"/>
```

接下来客户端实现,先绑定MessengerService服务,在根据服务端返回的Binder对象创建Messenger并使用此对象向服务端发送消息.

```java
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";

    private Messenger mMessenger;

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mMessenger = new Messenger(service);
            Message msg = Message.obtain(null, Constants.TYPE_MSG_FROM_CLIENT);
            Bundle bundle = new Bundle();
            bundle.putString("msg", "this is client msg");
            msg.setData(bundle);
            try {
                mMessenger.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        bindService(new Intent(this, MessengerService.class), mConnection, BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
    }
}
```

最后运行,看一下日志,很显然服务端收到了客户端的消息

```
01-18 09:49:01.823 31013-31013/? I/MessengerService: receiver client msg this is client msg
```

通过上面栗子可以看出,**Messenger中进行数据传输必须将数据放入Message中**,而Messenger和Message都实现了Parcelable接口,因此可以跨进程传输.简单来说Message中所支持的数据类型就是Messenger支持的数据类型,而Message中能作为载体传送数据的只有what,arg1,arg2,obj,replyTo,而obj在同一进程中是很实用的,但是进程间通讯的时候,在Android2.2以前obj不支持跨进程传递,2.2以后仅仅支持系统实现的Parcelable接口的对象才能通过它来传递,也就等于我们自定义的类即使实现了parcelable也无法通过obj传递,但是不要方,我们还有Bundle可以支持大量的数据传递.

具体在Message的obj字段的注释可以窥探一二.

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29message_obj.png)

可以看到在跨进程传递的时候,obj只支持非空的系统实现Parcelable接口的数据,要想传递其他数据使用setData,也就是Bundle方式,Bundle中可以支持大量的数据类型.

上面只能客户端向服务端发送信息,但有的时候我们还需要能够回应客户端,下面就介绍如何实现这种效果.还是上面的栗子只是稍微改下,当服务端接受到客户端消息后回复客户端接受成功.

首先我们修改下客户端,为了接受服务端发送消息,客户端也需要一个接受消息的Messenger和Handler

```java
private Messenger clientMessenger = new Messenger(new ClientHandler());

private static class ClientHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case Constants.TYPE_MSG_FROM_SERVICE:
                Log.i(TAG, msg.getData().getString("reply"));
                break;
        }
    }
}
```

还有一点就是客户端发送消息的时候,需要把接受服务端回复的messenger通过message的reply带到服务端

```java
private ServiceConnection mConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        mMessenger = new Messenger(service);
        Message msg = Message.obtain(null, Constants.TYPE_MSG_FROM_CLIENT);
        Bundle bundle = new Bundle();
        bundle.putString("msg", "this is client msg");
        msg.setData(bundle);
        msg.replyTo = clientMessenger;
        try {
            mMessenger.send(msg);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {

    }
};
```

然后是服务端的修改,需要修改MessengerHandler,当收到消息后,立即给客户端回复

```java
private static class MessengerHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case Constants.TYPE_MSG_FROM_CLIENT:
                Log.i(TAG, "receiver client msg " + msg.getData().getString("msg"));

                Message serviceMsg = Message.obtain(null, Constants.TYPE_MSG_FROM_SERVICE);
                Bundle bundle = new Bundle();
                bundle.putString("reply", "收到了");
                serviceMsg.setData(bundle);
                try {
                    msg.replyTo.send(serviceMsg);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                break;
        }
    }
}
```

运行后查看日志

```
01-18 14:00:13.873 29745-29745/? I/MessengerService: receiver client msg this is client msg

01-18 14:00:13.877 29715-29715/? I/MainActivity: 收到了
```

到这里Messenger进程间通讯介绍完了,这里给出一张Messenger工作原理图方便理解

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29messenger%E5%8E%9F%E7%90%86%E5%9B%BE.png)

栗子完了,该开车了,后面的赶紧的上车
现在我们从Messenger源码看看为啥说他是AIDL实现的,我们先随着服务端Messenger流程来分析.

1. 创建Messenger在构造方法中传入Handler

2. 在onBind中返回mMessenger.getBinder()

那么这里我们先看到Messenger参数为Handler的构造方法

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%8A%29Messenger0.png)

可以看到mTarget = target.getIMessenger()
那我们来到Handler中查看getIMessenger()方法

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29Messenger1.png)

可以发现返回值为MessengerImpl,那我们再来看MessengerImpl,发现他就是Handler中一个内部类

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29Messenger2.png)

看到IMessenger.Stub类后不知道有没有想到点什么,我稍微提示下在IPC上中我们使用AIDL的时候是不是在服务中写一个内部类实现了xxx.Stub,这里其实是一个套路等于Handler中有一个实现好的类MessengerImpl实现了send方法.如果你实在感觉不出啥这里附上[AIDL简单使用](http://blog.csdn.net/zly921112/article/details/53560913)链接仔细看看AIDL使用的流程,应该就能体会到.

那么看到现在我们已经知道mTarget就是MessengerImpl,也就是我们AIDL使用时候在服务端实现xxx.Stub类,然后在看我们在onBind中返回的mMessenger.getBinder()

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29Messenger3.png)

再往下追我没找到MessengerImpl的asBinder方法,但是猜也能猜到asBinder就是返回的xxx.Stub而Stub继承的Binder你要问我为啥能猜到你可以在看看[IPC(上)](http://blog.csdn.net/zly921112/article/details/54089500)中2.3.3Binder这一节关于AIDL生成类的分析

所以Service中Messenger的过程分析完发现其实跟AIDL几乎一模一样

- 创建一个Messenger就等于AIDL中实现xxx.stub,

- onBind中返回mMessenger.getBinder()就等于AIDL中在onBind返回我们实现的xxx.Stub类

再来根据客户端流程的分析

1. 绑定服务端,用服务端返回的IBinder对象创建一个Messenger

2. 然后用Messenger向服务端发送数据

绑定服务端跟aidl一样不做分析,用服务端返回的IBinder对象创建一个Messenger

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29Messenger4.png)

耶嘿,是不是感觉又似曾相识,没错AIDL中我们是xxx.Stub.asInterface(service)拿到服务端的接口,这里我们在Messenger构造方法中通过IMessenger.Stub.asInterface(target)拿到Handler替我们实现好的MessengerImpl

然后调用send发送数据,

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29Messenger5.png)

![](http://of1ktyksz.bkt.clouddn.com/ipc%28%E4%B8%AD%29Messenger2.png)

这个还是跟AIDL完全一致调用服务端实现接口的方法发送数据.
经过上面分析应该能完全明白为啥说Messenger底层是AIDL实现.

# 1.4  使用AIDL

在上面我们介绍了Messenger来进行进程间通信,可以发现Messenger是串行的方式处理客户端发来的消息,如果有大量消息同时发送到服务端,那么如果还是只能一个个处理就太不合适了,并且很多时候我们需要跨进程调用服务端方法,这时候用Messenger就无法做到了,但是我们可以使用AIDL来实现,AIDL也是Messenger的底层实现,因此Messenger本质上也是AIDL,只不过系统为我们做了封装方便调用而已.接下来介绍使用AIDL来进行进程间通信的流程,分为客户端和服务端.

1. 服务端
  首先创建一个Service来监听客户端的连接请求,然后创建一个AIDL文件,将要给客户端的接口在AIDL文件中声明,然后在Service实现AIDL文件生成的类.最后在onBind方法返回实现的类.

2. 客户端
  首先绑定服务端的Service,将连接成功后返回的Binder对象转换成AIDL接口所属类,接着就可以调用AIDL中的方法了.

上面描述的是AIDL的使用过程,在[IPC(上)](http://blog.csdn.net/zly921112/article/details/54089500)中我们已经讲过,这次我们会对其中的细节和难点进行详细的介绍.并完善在IPC(上)Binder那一节中提供的栗子.

## 1  AIDL接口的创建

首先看AIDL接口的创建,如下所示,我们创建了一个后缀为AIDL的文件,在里面声明了一个接口和两个方法

```
// IBookManager.aidl
package com.zly.www.ipc2;

// Declare any non-default types here with import statements
import com.zly.www.ipc2.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}

```

在AIDL中并不是所有类型都可以使用,具体可以使用的类型如下

- 基本数据类型(int,long,char,boolean,double等)

- String 和 CharSequence;

- List 中的所有元素都必须是以上列表中支持的数据类型、其他 AIDL 生成的接口或您声明的可打包类型。 可使用List泛型（例如，List<String>）。另一端实际接收的具体类始终是 ArrayList，生成的方法中支持的类型是 List 接口。

- Map 中的所有元素都必须是以上列表中支持的数据类型、其他 AIDL 生成的接口或您声明的可打包类型。 不支持Map泛型（如 Map<String,Integer> 形式的 Map）。 另一端实际接收的具体类始终是 HashMap，生成的方法中支持的类型是 Map 接口。

- Parcelable: 所有实现了Parcelable接口的对象

- AIDL: 所有的AIDL接口本身也是可以在AIDL文件中使用的

以上6种数据类型就是AIDL所支持的所有类型,其中**自定义的Parcelable对象和AIDL对象必须要显示的import进来**,不管他们是否和当前AIDL文件位于同一包内.

另一个需要注意的地方是,如果AIDL文件中使用了自定义的类,那么它必须继承Parcelable,因为Android系统可通过它将对象分解成可编组到各进程的原语.并且必须新建一个和它同名的AIDL文件,并在其中声明它为Parcelable类型,在上面IBookManager.aidl中我们用到了Book这个类,所以我们需要创建Book.aidl,并添加如下内容

```
package com.zly.www.ipc2;
parcelable Book;
```

除此在外,AIDL中除了基本数据类型,其他类型的参数必须标上方向:in,out或者inout,in表示输入型参数,out表示输出型参数,inout表示输入输出型参数.至于区别有点长下一段讲解.我们要根据实际需要去指定类型,不能一概使用out或者inout,因为这在底层有开销,最后AIDL接口中只支持方法,不支持声明静态常亮,这一点有别于传统接口.

接下来解释in,out,inout的意义
AIDL中的定向 tag 表示了在跨进程通信中数据的流向，其中 in 表示数据只能由客户端流向服务端， out 表示数据只能由服务端流向客户端，而 inout 则表示数据可在服务端与客户端之间双向流通。其中，数据流向是针对在客户端中的那个传入方法的对象而言的。in 为定向 tag 的话表现为服务端将会接收到一个那个对象的完整数据，但是客户端的那个对象不会因为服务端对传参的修改而发生变动；out 的话表现为服务端将会接收到那个对象的的空对象，但是在服务端对接收到的空对象有任何修改之后客户端将会同步变动；inout 为定向 tag 的情况下，服务端将会接收到客户端传来对象的完整信息，并且客户端将会同步服务端对该对象的任何变动。其实具体的原因在AIDL生成的类中一看便知这里不再赘述.

为了方便AIDL开发,建议把所有和AIDL相关的类和文件放入一个包,这样把整个包复制到客户端比较方便.需要注意的是,AIDL的包结构在服务端和客户端要保持一致,否则会出错,因为客户端需要反序列化服务端中的AIDL接口相关的所有类,如果类的完整路径不一样的话,就无法成功反序列化,程序也就无法正常的运行.

## 2  远程服务端Service实现

接下来我们就要实现AIDL接口了,代码如下

```java
public class BookManagerService extends Service {

    private static final String TAG = "BookManagerService";

    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();

    private Binder mBinder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "Ios"));
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }


}

```

首先在onCreate中初始化添加了两本图书的信息,然后创建一个Binder对象并在onBind中返回它,这个对象继承自IBookManager.Stub并实现了它内部的AIDL方法,这个过程之前讲过这里不再介绍,注意这里采用了CopyOnWriteArrayList,这个CopyOnWriteArrayList支持并发读/写.在Binder那节我们说过,AIDL方法是在服务的Binder线程池中执行的,因此在多个客户端同时连接的时候,会存在多个线程同时访问的情况,所以我们要在AIDL中处理线程同步,而我们这里直接使用CopyOnWriteArrayList来进行自动的线程同步.

前面我们说过,AIDL中能够使用的List只有ArrayList,但这里我们使用了CopyOnWriteArrayList(它不是继承的ArrayList),为什么可以正常工作,这个因为AIDL中所支持的是抽象的List接口,因此虽然服务端返回的是CopyOnWriteArrayList,但在Binder中会按照List的规范去访问数据并最终形成一个新的ArrayList传递给客户端,所以我们在服务端采用CopyOnWriteArrayList是完全可以.

现在我们需要注册BookManager让它运行在独立的进程中.

```
<service android:name=".BookManagerService"
    android:process=":remote"/>
```

## 3  客户端的实现
首先绑定远程服务,绑定成功后将服务端返回的Binder对象转换成AIDL接口,然后就可以通过这个接口调用服务端的远程方法了,代码如下

```java
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);

            try {
                List<Book> list = bookManager.getBookList();
                Log.i(TAG, "query book list, list type:" + list.getClass().getCanonicalName());
                Log.i(TAG, "query book list:" + list.toString());
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        bindService(new Intent(this, BookManagerService.class), mConnection, BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
    }
}
```

绑定成功后,会通过bookManager调用getBookList方法,然后打印获得的图书信息.这里有一点需要注意,服务端的方法可能需要很久才能执行完毕,所以在UI线程执行就有可能ANR,这里之所以这么写是为了方便演示.

运行后日志如下

```
01-20 08:28:36.341 13073-13073/com.zly.www.ipc2 I/MainActivity: query book list, list type:java.util.ArrayList
01-20 08:28:36.341 13073-13073/com.zly.www.ipc2 I/MainActivity: query book list:[Book{bookId=1, bookName='Android'}, Book{bookId=2, bookName='Ios'}]
```

可以发现,虽然我们在服务端返回的是CopyOnWriteArrayList类型,但是客户端收到的仍然是ArrayList类型,这也证实了我们前面所说的另一端接受的实际类型始终是ArrayList,第二行说明客户端成功得到了服务端的信息.

这就已经是一次完整的AIDL进行IPC的过程,但是还没完下面继续介绍AIDL中常见的难点,我们接着调用另一个方法addBook,我们在客户端添加一本书,然后在获取一次,看看程序是否正常工作.还是上面的代码,客户端在服务连接后,在onServiceConnected中做如下改动

```java
IBookManager bookManager = IBookManager.Stub.asInterface(service);

try {
    List<Book> list = bookManager.getBookList();
    Log.i(TAG, "query book list:" + list.toString());

    Book newBook = new Book(3, "android精通到跑路");
    bookManager.addBook(newBook);
    Log.i(TAG, "add book:" + newBook.toString());

    List<Book> newList = bookManager.getBookList();
    Log.i(TAG, "query book list:" + newList.toString());

} catch (RemoteException e) {
    e.printStackTrace();
}
```

很显然我们成功的向服务端添加了一本Android从精通到跑路

```
/MainActivity: query book list:[Book{bookId=1, bookName='Android'}, Book{bookId=2, bookName='Ios'}]
01-20 09:10:24.345 26474-26474/? I/MainActivity: add book:Book{bookId=3, bookName='android精通到跑路'}
01-20 09:10:24.345 26474-26474/? I/MainActivity: query book list:[Book{bookId=1, bookName='Android'}, Book{bookId=2, bookName='Ios'}, Book{bookId=3, bookName='android精通到跑路'}]
```

现在我们增加需求的难度,用户不想时不时的查询图书列表了,于是,他去问图书馆能不能有新书直接告诉我.这个时候应该能马上想到,这是一种典型的观察者模式,每个感兴趣的用户都观察新书,当新书到的时候,图书馆就通知每个对这本书感兴趣的用户.下面我们就这个情况来模拟,首先我们需要一个AIDL接口,每个用户都实现这个接口并且有图书馆提醒新书到了的功能.之所以选择AIDL接口而不是普通接口,是因为AIDL中不支持普通接口,这里我们创建一个IOnNewBookArrivedListener.aidl文件,我们期望的是,当服务端有新书来的时候,通知所有申请提醒功能的用户,从程序上说就是调用所有IOnNewBookArrivedListener对象中的OnNewArrived方法,并把新书作为参数传递给客户端.

```
// IOnNewBookArrivedListener.aidl
package com.zly.www.ipc2;

// Declare any non-default types here with import statements
import com.zly.www.ipc2.Book;
interface IOnNewBookArrivedListener {
    void onNewBookArrived(in Book newbook);
}
```

除了新加AIDL接口,我们还要在原有IBookManager.aidl中添加两个方法分别是申请提醒和取消提醒,这里需要注意即使在同一个包中加入AIDL也是需要import语句的.例如下面的import com.zly.www.ipc2.IOnNewBookArrivedListener;

```
// IBookManager.aidl
package com.zly.www.ipc2;

// Declare any non-default types here with import statements
import com.zly.www.ipc2.Book;
import com.zly.www.ipc2.IOnNewBookArrivedListener;
interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
    void registerListener(IOnNewBookArrivedListener listener);
    void unregisterListener(IOnNewBookArrivedListener listener);
}
```

接着服务端Service实现也要稍微修改,主要是我们新加方法的实现,同时在BookManagerService中还开启了一个线程,每个5秒就像书库中增加一本新书并通知所有感兴趣用户,代码如下

```java
public class BookManagerService extends Service {
    private static final String TAG = "BookManagerService";

    private AtomicBoolean mIsServiceDestoryed = new AtomicBoolean(false);

    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();
    private CopyOnWriteArrayList<IOnNewBookArrivedListener> mListenerList = new CopyOnWriteArrayList<>();

    private Binder mBinder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }

        @Override
        public void registerListener(IOnNewBookArrivedListener listener) throws RemoteException {
            if (!mListenerList.contains(listener)) {
                mListenerList.add(listener);
                Log.i(TAG, "registerListener success");
            } else {
                Log.i(TAG, "already exists");
            }
            Log.i(TAG, "registerListener size:" + mListenerList.size());
        }

        @Override
        public void unregisterListener(IOnNewBookArrivedListener listener) throws RemoteException {
            if (mListenerList.contains(listener)) {
                mListenerList.remove(listener);
                Log.i(TAG, "unregister listener success");
            } else {
                Log.i(TAG, "no found, can not unregister");
            }
            Log.i(TAG, "unregisterListener current size:" + mListenerList.size());
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "Ios"));
        new Thread(new ServiceWorker()).start();
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        mIsServiceDestoryed.set(true);
    }

    private void onNewBookArrived(Book book) {
        mBookList.add(book);
        for (int i = 0; i < mListenerList.size(); i++) {
            IOnNewBookArrivedListener listener = mListenerList.get(i);
            Log.i(TAG, "onNewBookArrived, notify listener:" + listener);
            try {
                listener.onNewBookArrived(book);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }

    private class ServiceWorker implements Runnable {

        @Override
        public void run() {
            while (!mIsServiceDestoryed.get()) {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                int bookId = mBookList.size() + 1;
                Book newBook = new Book(bookId, "new book#" + bookId);
                onNewBookArrived(newBook);
            }
        }
    }
}

```

最后,我们还需要改下客户端代码,主要两个方面:首先客户端需要注册IOnNewBookArrivedListener到服务端,同时在Activity退出的时候注销,另一个,当有新书的时候,服务端会回调客户端的IOnNewBookArrivedListener对象的OnNewBookArrived方法,但是这个方法是在客户端的BInder线程池中执行的,因此,为了便于进行UI操作,我们需要一个Handler可以将其切换到客户端的主线程中去执行.客户端代码修改如下.

```java
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";
    private static final int MESSAGE_NEW_BOOK_ARRIVED = 1;

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MESSAGE_NEW_BOOK_ARRIVED:
                    Log.i(TAG, "receive new book :" + msg.obj);
                    break;
            }
        }
    };

    private IBookManager mRemoteBookManager;
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            mRemoteBookManager = bookManager;
            try {
                List<Book> list = bookManager.getBookList();
                Log.i(TAG, "query book list:" + list.toString());

                Book newBook = new Book(3, "android精通到跑路");
                bookManager.addBook(newBook);
                Log.i(TAG, "add book:" + newBook.toString());

                List<Book> newList = bookManager.getBookList();
                Log.i(TAG, "query book list:" + newList.toString());

                bookManager.registerListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mRemoteBookManager = null;
            Log.i(TAG, "binder died");
        }
    };

    private IOnNewBookArrivedListener mOnNewBookArrivedListener = new IOnNewBookArrivedListener.Stub() {
        @Override
        public void onNewBookArrived(Book newbook) throws RemoteException {
            mHandler.obtainMessage(MESSAGE_NEW_BOOK_ARRIVED, newbook).sendToTarget();
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        bindService(new Intent(this, BookManagerService.class), mConnection, BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mRemoteBookManager != null && mRemoteBookManager.asBinder().isBinderAlive()) {
            Log.i(TAG, "unregister listener:" + mOnNewBookArrivedListener);
            try {
                mRemoteBookManager.unregisterListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        unbindService(mConnection);
    }
}
```

运行程序,看下日志,客户端的确收到了服务端每隔5s一次的新书推送

```
01-20 13:57:41.676 365-365/com.zly.www.ipc2 I/MainActivity: receive new book :Book{bookId=4, bookName='new book#4'}
01-20 13:57:46.677 365-365/com.zly.www.ipc2 I/MainActivity: receive new book :Book{bookId=5, bookName='new book#5'}
```

但是到此还没有结束,AIDL远不止这么简单,目前还有些难点我们还未涉及到.接下来继续飙车.

从上面代码可以发现,当MainActivity关闭时,我们会在onDestory中去解除注册到服务端的listener,就相当于我们不在需要新书提醒了,那我们按back退出MainActivity在查看日志.

```
01-20 14:02:48.584 498-510/com.zly.www.ipc2:remote I/BookManagerService: no found, can not unregister
01-20 14:02:48.584 498-510/com.zly.www.ipc2:remote I/BookManagerService: unregisterListener current size:1
```

从上面日志可以看出,在解除注册过程中,服务端竟然无法找到我们之前注册的那个listener,但是我们在客户端注册和解除注册传递的明明是同一个对象,仔细想想你就会发现,其实这是必然的,这种解除注册的方式在日常开发中经常用到,但是在多进程的开发中却无法奏效,因为Binder会把客户端传递过来的对象重新转化成一个新的对象,虽然我们注册和注销都传的同一个对象,但别忘了对象是不能跨进程传递的,对象传输本质上都是反序列化的过程,这就是为什么AIDL中自定义对象都必须实现Parcelable接口的原因,那么到底我们该怎么办呢,答案是使用RemoteCallbackList.接下来详细分析.

RemoteCallbackList是系统专门提供的用于删除跨进程Listener的接口.RemoteCallbackList是一个泛型,支持管理任意的AIDL接口,这点从他的声明就可以看出,因为所有的AIDL接口都继承自IInterface接口,在前面Binder那节有讲过.

```java
public class RemoteCallbackList<E extends IInterface>
```

它的工作原理很简单,它内部有一个Map结构专门来保存所有的AIDL回调,这个Map的key是IBinder类型,value是Callback类型,如下

```java
ArrayMap<IBinder, Callback> mCallbacks
            = new ArrayMap<IBinder, Callback>();
```

其中Callback中封装了远程listener,当客户端注册listener的时候,它会把这个listener的信息存入mCallbacks中,其中key和value通过如下方式获得

```java
IBinder key = callback.asBinder();
Callback value = new Callback(callback, cookie);
```

到这里,应该明白了,虽然说多次跨进程传递客户端的同一个对象会服务端会生成不同对象,但是这些新生成对象有个共同点,就是他们底层的Binder对象是同一个,利用这个特点,我们就可以实现上面的功能.当客户端注销的时候,我们只需要遍历服务端所有listener,找出那个和注销listener具有相同Binder对象的服务端listener并把它删除掉,这就是RemoteCallbackList为我们做的事情.同时RemoteCallbackList还有一个很有用的功能,就是当客户端进程终止后,它能够自动移除客户端所注册的listener,另外,RemoteCallbackList内部实现了线程同步的功能,所以我们使用它来注册和注销时候,不需要做额外的线程同步,下面就来演示如何使用.

我们要对BookManagerService做一些修改,首先要创建一个RemoteCallbackList对象来代替之前的CopyOnWriteArrayList.

```java
    private RemoteCallbackList<IOnNewBookArrivedListener> mListenerList = new RemoteCallbackList<>();
```

然后修改registerListener和unregisterListener这两个接口的实现

```java
@Override
public void registerListener(IOnNewBookArrivedListener listener) throws RemoteException {
    mListenerList.register(listener);
}

@Override
public void unregisterListener(IOnNewBookArrivedListener listener) throws RemoteException {
    mListenerList.unregister(listener);
}
```

接下来修改onNewBookArrived方法,当有新书的时候我们就需要通知所有注册的listener

```java
private void onNewBookArrived(Book book) {
    mBookList.add(book);
    int n = mListenerList.beginBroadcast();
    for (int i = 0; i < n; i++) {
        IOnNewBookArrivedListener listener = mListenerList.getBroadcastItem(i);
        if(listener != null){
            try {
                listener.onNewBookArrived(book);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
    mListenerList.finishBroadcast();
}
```

BookManagerService修改完毕了,为了方便我们验证程序的功能,我们还需要添加一些log,在注册和注销后我们分别打印所有listener数量,如果正常的话那么注册后是1,注销后是0,我们再次运行下看看日志.可以发现RemoteCallback完全可以完成跨进程的注销功能.

```
01-20 15:20:29.090 28830-28842/? I/BookManagerService: registerListener, current size: 1
01-20 15:20:36.479 28830-28842/com.zly.www.ipc2:remote I/BookManagerService: unregisterListener, current size: 0
```

使用RemoteCallbackList,有一点需要注意,虽然名字带List但是我们无法像List一样去操作它,遍历RemoteCallbackList必须按照如下方式进行,其中beginBroadcast和finishBroadcast必须要配对使用.

```java
int n = mListenerList.beginBroadcast();
for (int i = 0; i < n; i++) {
    IOnNewBookArrivedListener listener = mListenerList.getBroadcastItem(i);
    if(listener != null){
        //do something
    }
}
mListenerList.finishBroadcast();
```

到此aidl基本使用介绍完成,但还有几点需要说明,我们知道,客户端调用远程服务端服务的方法,被调用的方法运行在服务端的Binder线程池中,同时客户端会被挂起,这个时候如果比较耗时,会导致客户端当前线程长时间阻塞,如果是ui线程就会ANR,因此当我们知道某个远程方法是耗时的,那么就要避免在客户端ui线程去访问远程方法,由于客户端onServiceConnected和onServiceDisconnected方法都运行在UI线程所以不能直接在里面调用服务端耗时的方法,另外由于服务端方法本身就运行在服务端的binder线程池中,所以服务端方法本身就可以执行大量耗时操作,这个时候切记不要在服务端方法中开线程去执行异步任务,除非有明确必要,下面我们稍微修改下服务端的getBookList方法,我们假定这个方法是耗时的那么我们可以这么做.

```java
@Override
public List<Book> getBookList() throws RemoteException {
    SystemClock.sleep(5000);
    return mBookList;
}
```

然后在客户端放一个按钮,点击就调用服务端的getBookList方法,多点击几次,客户端就ANR了.
为了避免出现ANR其实很简单,我们只需要在非ui线程调用即可

```java
public void invoke(View v) {
    new Thread(new Runnable() {
        @Override
        public void run() {
            if (mRemoteBookManager != null) {
                try {
                    mRemoteBookManager.getBookList();
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
    }).start();
}
```

同样,当服务端调用客户端listener中的方法时,被调用的方法也运行在Binder线程池中,只不过是客户端线程池.所以我们同样不可以在服务端中调用客户端耗时方法,比如刚刚栗子中BookManagerService的onNewBookArrived方法,如下所示,调用了客户端内部的IOnNewBookArrivedListener中的onNewBookArrived方法,如果客户端这个方法比较耗时的话,那么服务端中onNewBookArrived方法同样需要运行在非ui线程.

```java
private void onNewBookArrived(Book book) {
    mBookList.add(book);
    int n = mListenerList.beginBroadcast();
    for (int i = 0; i < n; i++) {
        IOnNewBookArrivedListener listener = mListenerList.getBroadcastItem(i);
        if(listener != null){
            try {
                listener.onNewBookArrived(book);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
    mListenerList.finishBroadcast();
}
```

另外,由于客户端IOnNewBookArrivedListener中的onNewBookArrived方法运行在客户端Binder线程池,所以它不能直接操作ui,如果要操作ui需要使用handler回到ui线程.

为了程序的健壮性,我们要需要一件事情,Binder可能意外死亡,这时我们需要重新连接服务,有两种方法一种是给Binder设置DeathRecipient监听,当Binder死亡时候我们会收到binderDied回调,在binderDied方法中我们可以重新连接远程服务.另一种是在onServiceDisconnected中重连远程服务.他们区别在于:onServiceDisconnected在UI线程被回调,而binderDied在客户端的Binder线程池中回调.下面我们验证下两者的区别.首先我们通过ddms杀死服务端进程,然后在这两个方法打印当前线程名称.

```
01-20 16:15:14.401 18827-18840/? I/MainActivity: binderDied :Binder_2
01-20 16:15:14.401 18827-18827/? I/MainActivity: onServiceDisconnected :main
```

从日志上可以看到,onServiceDisconnected运行在main线程,而binderDied运行在Binder_2线程,很显然是Binder线程池中的一个线程.

到此为此,我们已经对AIDL有一个系统的认识了,但还有一点,如何在AIDL中使用权限验证功能,默认情况下,我们的远程服务任何人都可以连接,但这不是我们愿意看到的,所以我们必须加入权限验证功能,权限验证失败则无法调用服务中的方法,这AIDL中进行权限验证,我们这里介绍两个方法.

第一种我们在onBind中进行验证,验证通不过就返回null,这样验证失败的客户端无法直接绑定服务,至于验证方式有很多种,比如使用permission验证.使用这种方式,我们需要在androidmenifest中声明所需的权限

```
<permission android:name="com.zly.www.ipc2.permission.ACCESS_BOOK_SERVICE"
    android:protectionLevel="normal"/>
```

关于permission的定义方式我转载了一篇blog,[Android自定义权限](http://blog.csdn.net/zly921112/article/details/54668655),定义了权限后,我们可以在BookManagerService的onBInd方法中做权限验证,如下所示

```java
public IBinder onBind(Intent intent) {
    int check = checkCallingOrSelfPermission("com.zly.www.ipc2.permission.ACCESS_BOOK_SERVICE");
    if(check == PackageManager.PERMISSION_DENIED){
        return null;
    }
    return mBinder;
}
```

这样一个应用来绑定服务的时候,会验证这个应用的权限,如果没有这个权限,onBInd方法就会返回null,最终无法绑定我们的服务,同样这种方式也适用于Messenger中.

如果我们自己内部应用想绑定我们的服务,需要在androidMenifest文件中采用如下方式使用

```
    <uses-permission android:name="com.zly.www.ipc2.permission.ACCESS_BOOK_SERVICE"/>
```

第二种方式,我们可以在服务端的onTransact方法进行权限验证,如果验证失败就返回false,这样服务端就不会终止AIDL中的方法达到保护服务端的效果.也可以采用permission方式.具体实现方式和第一种一样,只不过这里还加了包名验证.

```java
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws RemoteException {
            // 权限验证
            int check = checkCallingOrSelfPermission("com.example.test1.permission.ACCESS_BOOK_SERVICE");
            L.d("check:"+check);
            if(check==PackageManager.PERMISSION_DENIED){
                L.d("Binder 权限验证失败");
                return false;
            }
            // 包名验证
            String packageName=null;
            String[] packages = getPackageManager().getPackagesForUid(getCallingUid());
            if(packages!=null && packages.length>0){
                packageName = packages[0];
            }
            if(!packageName.startsWith("com.example")){
                L.d("包名验证失败");
                return false;

            }
            return super.onTransact(code, data, reply, flags);
        };
```

当然除了以上两种验证方式还可以为service指定permission等,这里不依依介绍了.

剩下的下一篇在写了...真心太多了..写的我要呕心沥血了...下面一篇在介绍IPC另外两种方式...

最后不能忘了谢鸣:中间看aidl文档的时候有了涅槃大胸弟的帮助很快的理解了原来generic是泛型不是通用...特此感谢...你问我是谁涅槃大胸弟,我只能告诉你胸很大的就是的.
