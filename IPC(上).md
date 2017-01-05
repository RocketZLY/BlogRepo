# 1 IPC介绍
既然是IPC的开篇那么先介绍下IPC的定义
IPC:进程间通信或者跨进程通信,即进程间交换数据的过程.
说到进程,那么需要了解下什么是进程.什么是线程,按操作系统描述,线程是CPU调度的最小单元,同时线程是一种有限的系统资源,而进程指一个执行单元,在PC和移动设备上指一个程序或者应用,一个进程可以包含多个线程,因此**进程和线程是包含和被包含的关系**,

在Android中进程间通信方式就是Binder了,除了Binder还有Socket不仅可以实现进程通信,也可以实现任意终端之间的通信,

IPC在多进程使用情况分为两种:  
- 应用因为某些原因自身需要采用多进程模式来运行,比如某些模块由于特殊原因需要运行在单独的进程中,又或者为了加大一个应用可使用的内存所以需要通过多进程来获取更多内存空间.  

- 应用需要访问其他应用的数据.甚至我们通过系统提的ContentProvider去查询数据的时候,也是一种进程间通信.

总之采用了多进程的设计方法,那么就必须解决进程间通信的问题了.

# 2 Android中多进程模式
在Android中通过给四大组件指定android:process属性,我们可以轻易的开启多进程模式,但是同时也会带来一个麻烦,下面将会一一介绍

## 2.1 开启多进程模式
Android中多进程一般指一个应用中存在多个进程的情况,因此这里不讨论两个应用之间的多进程情况,

一般情况下Android中使用多进程只有一个方法,就是给四大组件在AndroidMenifest中指定android:process属性,还有一个非常特殊的方法,就是通过JNI在native层去fork一个新的进程,由于不常用,所以这里只介绍一般情况下的创建方式

下面是一个例子描述如何创建多线程
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
      package="com.zly.www.ipc">

      <application
          android:allowBackup="true"
          android:icon="@mipmap/ic_launcher"
          android:label="@string/app_name"
          android:supportsRtl="true"
          android:theme="@style/AppTheme">
          <activity android:name="com.zly.www.ipc.MainActivity">
              <intent-filter>
                  <action android:name="android.intent.action.MAIN" />

                  <category android:name="android.intent.category.LAUNCHER" />
              </intent-filter>
          </activity>

          <activity android:name="com.zly.www.ipc.TwoActivity"
              android:process=":zly"/>

          <activity android:name="com.zly.www.ipc.ThreeActivity"
              android:process="com.zly.www.ipc.zly"/>
      </application>

  </manifest>


这里是三个activity.
MainActivity未指定process所以运行在默认的进程中,而默认的进程名是包名.TwoActivity和ThreeActivity指定了不同的process,所以将会新建两个进程TwoActivity process值为:zly,而":"的含义指当前进程名为 包名 + ":zly",是一种简写,所以进程为com.zly.www.ipc:zly. ThreeActivity process值为com.zly.www.ipc.zly所以进程名就是com.zly.www.ipc.zly

其次,进程名以:开头的进程为当前应用的私有进程,其他应用的组件不可以和它跑在同一进程中,而进程名不以:开头的属于全局进程,其他应用可以通过shareUID方式和他跑在同一进程.

这里可以通过ddms工具看出  
![](http://of1ktyksz.bkt.clouddn.com/ipc1/Image.png)  
确实开起了三个进程,但这只是开始,实际使用中多进程还有很多问题需要处理后面将细说

## 2.2 多进程模式的运行机制
如果用一句话来形容多进程,那么可以这样描述,当应用开启了多进程以后,各种奇怪的现象就都来了,举个栗子,新建一个类User,里面有一个静态成员变量uId如下

    public class UserBean {

      public static int uId = 1;

    }


其中MainActivity运行在默认进程,TwoActivity通过指定process运行在一个独立的进程中,然后在MainActivity的onCreate()方法中将uId赋值为2,打印出这个变量,再在TwoActivity中打印该值  
![](http://of1ktyksz.bkt.clouddn.com/ipc1/Image2.png)  
发现结果和我们平常的所见的完全不同,按正常情况下TwoActivity中uid应该也是2才对,看到这里也就应了前面所说的,当应用开启了多进程以后,各种奇怪的现象就都来了,所以多进程绝非仅仅只是添加一个process属性那么简单.

上述问题出现的原因是TwoActivity运行在一个单独的进程中,而Android为每一个应用分配了一个独立的虚拟机,或者说为每一个进程都分配一个独立的虚拟机,不同的虚拟机在内存分配上有不同的地址空间,这就导致在不同的虚拟机中访问同一个类的对象会产生多份副本.拿我们这个例子来说,在TwoActivity所在进程和MainActivity所在进程都存在一个UserBean类,并且这两个类互相不干扰,在一个进程中修改uId值只会影响当前进程,对其他进程都不会造成任何影响,这样就解释了为什么在MainActivity修改uId的值,而在TwoActivity中uId值没有发生改变.

**所有运行在不同进程中的四大组件,只要他们之间通过内存来共享数据都会失败**,这也是多进程带来的主要影响.一般来说,使用多进程会造成如下几个方面的问题
1. 静态成员变量和单例模式失效

2. 线程同步机制完全失效

3. SharedPreferences的可靠性下降

4. Application会多次创建

第1个问题上面已经分析了.第2个问题本质和第1个是一个问题,既然都不是一个内存了,那么不管是锁对象还是锁全局类都无法保证线程同步,因为不同进程锁的不是同一个对象.第3个问题是因为SharedPreferences不支持两个进程同时执行操作否则会有数据丢失的可能,第4个问题,当一个组件跑在一个新的进程中的时候,由于系统要在创建新的进程同时分配独立虚拟机,所以这个过程其实就是启动一个应用的过程.因此,相当于系统又把这个应用重新启动了一遍,既然重启了当然会创建新的application,其实可以这么理解,**运行在同一个进程的组件是属于同一个虚拟机和同一个application**.为了更加清晰的展示着一点,下面我们做个测试,首先在Application的onCreate方法中打印出当前进程的名字,然后连续启动三个同一个应用内但属于不同进程的Activity,按照期望,Application的onCreate应该执行三次,并且打印出来的三次的进程名不同.  

    public class AppApplication extends Application {

        @Override
        public void onCreate() {
            super.onCreate();
            Logger.init("zhuliyuan");

            Logger.d(getCurProcessName(this));
        }

        String getCurProcessName(Context context) {
            int pid = android.os.Process.myPid();
            ActivityManager mActivityManager = (ActivityManager) context
                    .getSystemService(Context.ACTIVITY_SERVICE);
            for (ActivityManager.RunningAppProcessInfo appProcess : mActivityManager
                    .getRunningAppProcesses()) {
                if (appProcess.pid == pid) {

                    return appProcess.processName;
                }
            }
            return null;
        }
    }

运行以后日志如下  
![](http://of1ktyksz.bkt.clouddn.com/ipc1/QQ%E6%88%AA%E5%9B%BE20161016154949.png)
可以看出,Application执行了三次onCreate,并且每次进程名都不一样,它的进程名为我们在Manifest文件中指定的process属性值一致.这也就证明了再多进程模式中,**不同进程的组件的确会拥有独立的虚拟机,Application和内存空间**,这会给我们开发带来困扰,尤其需要注意.

虽然多进程会给我们带来很多问题,但是系统给我们提供了很多跨进程通讯的方法,虽然说不能直接共享内存,但是通过跨进程通讯我们还是可以实现数据交互.实现跨进程通讯的方式有很多,比如Intent传递数据,共享文件和SharedPreferences,基于Binder的Messager和AiDL以及Socket,但是为了更好的理解IPC各种方式,下面先介绍些基础概念.

## 2.3 IPC基础概念
下面介绍IPC中一些基本概念,Serializable接口,Parcelable接口以及Binder,熟悉这三种方式后,后面才能更好的理解跨进程通信的各种方式.

## 2.3.1 Serializable接口
Serializable接口是Java提供的一个序列化接口,它是一个空接口,为对象提供标准的序列化和反序列化操作,实现Serializable来实现序列化相当简单,只需要在类的声明中指定一个类似下面的标示即可自动实现默认的序列化过程.

    public class UserBean implements Serializable{

        private static final long serialVersionUID = 213213213123L;

        public static int uId = 1;
    }
当然serialVersionUID并不是必须的,我们不声明这个同样也可以实现序列化,但是这个会对反序列化产生影响,具体影响后面介绍.

通过Serializable实现对象的序列化,非常简单,几乎所有的工作都被系统完成了,具体对象序列化和反序列化如何进行的,只需要采用ObjectOutputStream和ObjectInputStream即可实现,下面看一个栗子

    //序列化
    User user = new User("大帅比");
    ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(file));
    oos.writeObject(bean);
    oos.close();

    //反序列化过程
    ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file));
    User user = (User)ois.readObject();
    ois.close;

当然还可以通过重写writeObject()与readObject()自定义序列化过程.

    private void writeObject(java.io.ObjectOutputStream out)
        throws IOException {
      // write 'this' to 'out'...
    }

    private void readObject(java.io.ObjectInputStream in)
        throws IOException, ClassNotFoundException {
      // populate the fields of 'this' from the data in 'in'...
    }

上述代码演示了采用Serializable方式序列化对象的过程,有一点需要注意的是恢复后user和之前的user并不是同一个对象,这里我转载了一篇关于serializable序列化的blog有兴趣的小伙伴可以看看[Java序列化与反序列化](http://blog.csdn.net/zly921112/article/details/52854102).

刚开始提到,即使不指定serialVersionUID也可以实现序列化,那么这个到底是干嘛的呢,其实这个serialVersionUID是用来辅助序列化和反序列化过程的,**原则上序列化后的数据中的serialVersionUID只有和当前类的serialVersionUID相同才能够正常的被反序列化**.serialVerisonUID的工作机制是这样:序列化的时候把当前类的serialVersionUID写入序列化文件中(也可能是其他中介),当反序列化的时候系统会去检测文件中的SerialVerisonUID,看是否和当前类一致,如果一致则说明序列化的类和反序列化的类的版本相同序列化成功,否则说明当前类和序列化时的类发生了某些变化,这个时候无法序列化,会报如下异常

    java.io.InvalidClassException: com.zly.www.ipc.UserBean;
    Incompatible class (SUID): com.zly.www.ipc.UserBean:static final long serialVersionUID =213213213123L;
    but expectedcom.zly.www.ipc.UserBean: static final long serialVersionUID =21333123L;

一般来说,我们应该手动指定serialVersionUID的值,如果不手动指定serialVersionUID,反序列化时当前类有所改变,比如增加或者删除了某些成员变量,那么系统会重新计算当前hash值并把他赋给serialVersionUID,这个时候当前类serialVerisonUID就和当前类不一致,序列化失败,所以当我们手动指定了它以后,就可以很大程度避免反序列化失败.比如当版本升级以后,我们可能只删除或者新增了某些成员变量,这个时候我们反序列化仍然能成功,程序仍然能最大限度的恢复数据,相反,如果不指定serialVerisonUID的话,程序则会crash,当然如果类的结构发生了非常规性改变,比如修改了类名,修改了成员变量的类型,这个时候尽管serialVersionUID通过了,但是反序列化的过程还是会失败,因为类的结构发生了毁灭性改变.

另外需要注意下,首先静态成员变量属于类不属于对象,所以不会参与序列化过程;其次用transient关键字标记的成员变量不参与序列化过程.

## 2.3.2 Parcelable接口
Parcelable也是一个接口,只要实现这个接口,一个类和对象就可以实现序列化并可以通过Intent和Binder传递.下面是个典型用法.

    public class UserBean implements Parcelable{


        public static int uId = 1;

        public String userName;

        public int age;


        protected UserBean(Parcel in) {
            userName = in.readString();
            age = in.readInt();
        }

        @Override
        public void writeToParcel(Parcel dest, int flags) {
            dest.writeString(userName);
            dest.writeInt(age);
        }

        @Override
        public int describeContents() {
            return 0;
        }

        public static final Creator<UserBean> CREATOR = new Creator<UserBean>() {
            @Override
            public UserBean createFromParcel(Parcel in) {
                return new UserBean(in);
            }

            @Override
            public UserBean[] newArray(int size) {
                return new UserBean[size];
            }
        };
    }
在序列化的过程中需要实现功能有序列化,反序列化和内容描述.序列化功能由writeToParcel方法完成,是通过Parcel中一系列write方法来完成,反序列化功能由CREATOR来完成,其内部表明了如何创建序列化对象和数组,并通过Parcel一系列read方法在完成反序列化,内容描述由describeContents方法完成.

既然Parcelable与Serializable都能实现序列化,那我们该如何选择呢,Parcelable主要用在内存序列化上,其他情况用Serializable

## 2.3.3 Binder
由于Binder是一个非常深入的话题不是一两句能说清的,所以本节侧重Binder的使用和上层原理.

简单的说,Binder是Android中的一个类,实现了IBinder接口.Android开发中,Binder主要用于Service中,包括AIDL和Messenger,其中普通service中的Binder不涉及进程间通信,而Messenger底层就是AIDL,所以这里从AIDL入手分析Binder工作机制.(AIDL不太熟悉的同学可以参考我另一篇blog回顾下 [AIDL使用](http://blog.csdn.net/zly921112/article/details/53560913))

这里新建一个AIDL示例,然后SDK会自动为我们生成AIDL所对应的Binder类,然后我们可以分析Binder工作过程.创建自定义数据类型Book.java和自定义类型说明文件Book.aidl和IBookManager.aidl代码如下

    //Book.java
    public class Book implements Parcelable {
        public int bookId;
        public String bookName;

        public Book(int bookId, String bookName) {
            this.bookId = bookId;
            this.bookName = bookName;
        }

        protected Book(Parcel in) {
            bookId = in.readInt();
            bookName = in.readString();
        }

        public static final Creator<Book> CREATOR = new Creator<Book>() {
            @Override
            public Book createFromParcel(Parcel in) {
                return new Book(in);
            }

            @Override
            public Book[] newArray(int size) {
                return new Book[size];
            }
        };

        @Override
        public int describeContents() {
            return 0;
        }

        @Override
        public void writeToParcel(Parcel dest, int flags) {
            dest.writeInt(bookId);
            dest.writeString(bookName);
        }
    }

    //Book.aidl
    package com.zly.www.ipcdemo;
    parcelable Book;

    //IBookManager.aidl
    package com.zly.www.ipcdemo;

    // Declare any non-default types here with import statements
    import com.zly.www.ipcdemo.Book;

    interface IBookManager {

        List<Book> getBookList();

        void addBook(in Book book);

    }

上面三个文件中,Book.java是图书信息类,实现了Parcelable接口,Book.aidl是book在AIDL中的声明,IBookManager是我们定义的接口.里面有两个方法getBookList和addBook,这里这两个方法主要用于示例.下面看下系统根据IBookManager.aidl生成的Binder类,在app\build\generated\source\aidl\debug\包名 目录下有一个IBookManager.java类,接下来我们要根据这个生成的类分析Binder工作原理.

    /*
     * This file is auto-generated.  DO NOT MODIFY.
     * Original file: C:\\Users\\Administrator\\Desktop\\IPCDemo\\app\\src\\main\\aidl\\com\\zly\\www\\ipcdemo\\IBookManager.aidl
     */
    package com.zly.www.ipcdemo;

    public interface IBookManager extends android.os.IInterface {
        /**
         * Local-side IPC implementation stub class.
         */
        public static abstract class Stub extends android.os.Binder implements com.zly.www.ipcdemo.IBookManager {
            private static final java.lang.String DESCRIPTOR = "com.zly.www.ipcdemo.IBookManager";

            /**
             * Construct the stub at attach it to the interface.
             */
            public Stub() {
                this.attachInterface(this, DESCRIPTOR);
            }

            /**
             * Cast an IBinder object into an com.zly.www.ipcdemo.IBookManager interface,
             * generating a proxy if needed.
             */
            public static com.zly.www.ipcdemo.IBookManager asInterface(android.os.IBinder obj) {
                if ((obj == null)) {
                    return null;
                }
                android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
                if (((iin != null) && (iin instanceof com.zly.www.ipcdemo.IBookManager))) {
                    return ((com.zly.www.ipcdemo.IBookManager) iin);
                }
                return new com.zly.www.ipcdemo.IBookManager.Stub.Proxy(obj);
            }

            @Override
            public android.os.IBinder asBinder() {
                return this;
            }

            @Override
            public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
                switch (code) {
                    case INTERFACE_TRANSACTION: {
                        reply.writeString(DESCRIPTOR);
                        return true;
                    }
                    case TRANSACTION_getBookList: {
                        data.enforceInterface(DESCRIPTOR);
                        java.util.List<com.zly.www.ipcdemo.Book> _result = this.getBookList();
                        reply.writeNoException();
                        reply.writeTypedList(_result);
                        return true;
                    }
                    case TRANSACTION_addBook: {
                        data.enforceInterface(DESCRIPTOR);
                        com.zly.www.ipcdemo.Book _arg0;
                        if ((0 != data.readInt())) {
                            _arg0 = com.zly.www.ipcdemo.Book.CREATOR.createFromParcel(data);
                        } else {
                            _arg0 = null;
                        }
                        this.addBook(_arg0);
                        reply.writeNoException();
                        return true;
                    }
                }
                return super.onTransact(code, data, reply, flags);
            }

            private static class Proxy implements com.zly.www.ipcdemo.IBookManager {
                private android.os.IBinder mRemote;

                Proxy(android.os.IBinder remote) {
                    mRemote = remote;
                }

                @Override
                public android.os.IBinder asBinder() {
                    return mRemote;
                }

                public java.lang.String getInterfaceDescriptor() {
                    return DESCRIPTOR;
                }

                @Override
                public java.util.List<com.zly.www.ipcdemo.Book> getBookList() throws android.os.RemoteException {
                    android.os.Parcel _data = android.os.Parcel.obtain();
                    android.os.Parcel _reply = android.os.Parcel.obtain();
                    java.util.List<com.zly.www.ipcdemo.Book> _result;
                    try {
                        _data.writeInterfaceToken(DESCRIPTOR);
                        mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                        _reply.readException();
                        _result = _reply.createTypedArrayList(com.zly.www.ipcdemo.Book.CREATOR);
                    } finally {
                        _reply.recycle();
                        _data.recycle();
                    }
                    return _result;
                }

                @Override
                public void addBook(com.zly.www.ipcdemo.Book book) throws android.os.RemoteException {
                    android.os.Parcel _data = android.os.Parcel.obtain();
                    android.os.Parcel _reply = android.os.Parcel.obtain();
                    try {
                        _data.writeInterfaceToken(DESCRIPTOR);
                        if ((book != null)) {
                            _data.writeInt(1);
                            book.writeToParcel(_data, 0);
                        } else {
                            _data.writeInt(0);
                        }
                        mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                        _reply.readException();
                    } finally {
                        _reply.recycle();
                        _data.recycle();
                    }
                }
            }

            static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
            static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        }

        public java.util.List<com.zly.www.ipcdemo.Book> getBookList() throws android.os.RemoteException;

        public void addBook(com.zly.www.ipcdemo.Book book) throws android.os.RemoteException;
    }

上述代码就是系统生成的IBookManager.java,他继承了IInterface接口,同时他自己也是一个接口,所有在Binder中传输的接口都需要继承IInterface这个接口.虽然看起来有点混乱,但实际还是清晰的,首先,他声明了两个方法getBookList和addBook,显然这就是我们在IBookManager.aidl中声明的方法,同时它还声明了两个整形的id TRANSACTION_getBookList与TRANSACTION_addBook用于标识这两个方法,这两个标识用于在transact过程中区分客户端到底请求的哪个方法,然后他有个内部类Stub,Stub继承了Binder,当客户端和服务端都位于一个进程时,方法调用不会走跨进程的transact过程.当两者位于不同进程时,方法调用需要走transact过程,这个逻辑由Stub内部代理Proxy来完成,可以发现IBookManager这个接口很简单,这个接口实现的核心就是它的内部类Stub和Stub内部代理类Proxy,下面具体介绍下每个方法含义:

- DESCRIPTOR

  Binder唯一标识,一般为当前Binder的类名.

- asInterface(android.os.IBinder obj)

  用于将服务端的Binder对象转换成客户端所需的AIDL接口对象,这种转换过程区分进程,如果客户端和服务端在同一进程,那么此方法返回的就是服务端的Stub对象本身,否则就返回的系统封装后的Stub.proxy对象.

  这里我们来分析下,一般AIDL我们是在service中写一个内部类这里暂且叫MyBinder实现Stub,然后在客户端我们根据Stub.asInterface()获取service中的实现MyBinder,在调用他的方法....
  在service中我们仅仅是实现了Stub并未做其他操作,所以这个流程中能够决定我们究竟是走跨进程的transact方法还是直接走MyBinder实现的方法关键就看asInterface,接下来看代码

  ![](http://of1ktyksz.bkt.clouddn.com/ipc1_image3.png)

  可以看到有两种不同的返回值,一个是获取到的android.os.IInterface,一个是Stub.Proxy对象,这里先分析android.os.IInterface为返回值的时候

  ![](http://of1ktyksz.bkt.clouddn.com/ipc1_image5.png)

  它先判断唯一标识DESCRIPTOR是否相同如果相同返回mOwner,再来看看mOwner是什么

  ![](http://of1ktyksz.bkt.clouddn.com/ipc1_image6.png)
  ![](http://of1ktyksz.bkt.clouddn.com/ipc1_image7.png)

  可以看到mOwner是通过attachInterface()方法传入的,而Stub在构造方法的时候将this传入,所以mOwner就是Stub,也就是我们service中实现的MyBinder对象,所以当返回值为android.os.IInterface的时候我们实际就走的本地的

  再来看Stub.Proxy为返回值的时候,当我们拿到proxy对象在调用他的getBookList或者addBook方法的时候都走了transact方法,通过下面代码可以看出

  ![](http://of1ktyksz.bkt.clouddn.com/ipc1_image4.png)

  然后transact方法实际调用的

  ![](http://of1ktyksz.bkt.clouddn.com/ipc1_image9.png)

  OnTransact方法我们在Stub重写了

  ![](http://of1ktyksz.bkt.clouddn.com/ipc1_image10.png)

  经过这个流程就回到服务端的实现的Stub了,所以当返回值为Stub.Proxy我们实际走的跨进程的

  当然这里还有些疑问比如Binder是如何从service传递到client的,然后他为何调用了transact就为跨进程调用..目前我也不知道,但是开发艺术在后面应该会讲,所以我们暂且把这个问题留下后面再解决

- asBinder

  用于返回当前Binder对象

- onTransact

  这个方法运行在服务端中的Binder线程池中,当客户端发起跨进程请求时,远程请求会通过系统底层封装后调用此方法,该方法的原型为onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags),服务端通过code可以确定客户端所请求的是哪个方法,接着从data中取出目标方法需要的参数(如果目标方法有参数的话),然后执行目标方法,当方法执行完以后,就会向reply中写入返回值(如果目标方法有返回值的话),需要注意的是,如果此方法返回false,那么客户端会请求失败,因此我们可以利用这特性来做权限验证.

- Proxy#getBookList

  这个方法运行在客户端,当客户端远程调用此方法时,它的内部实现是这样的,先创建该方法需要的输入型Parcel对象_data,输出型Parcel对象_reply和返回值对象List;然后把该方法的参数信息写入_data中(如果有参数的话);接着调用transact方法来发起RPC(远程过程调用),同时当前线程挂起,然后服务端的onTransact方法会被调用,直到RPC过程返回后,当前线程继续执行,并从_reply中取出RPC过程的返回结果,最后返回_reply中的数据

- Proxy#addBook

  这个方法运行在客户端,它执行过程和getBookList一样,不过addBook没有返回值,所以他不需要从_reply中取出返回值.

通过上面的分析,我们应该对Binder工作流程有了个简单的了解,但是还有两点需要说明下.

1. 当客户端发起远程请求的时候,由于当前线程会被挂起直至服务端进程返回,所以如果远端操作是一个很耗时,那么不能再ui线程中发起远程请求

2. 由于服务端的Binder方法运行在Binder的线程池中,所以Binder方法不管是否耗时都应该采用同步的方式去实现,因为它已经运行在一个线程中了.

下面给出一个binder的工作机制图,方便理解

![](http://of1ktyksz.bkt.clouddn.com/ipc1_image11.png)

通过上面的分析来看,我们完全可以不写AIDL文件实现Binder,之所以写AIDL文件,是为了系统帮我们生成代码,系统根据AIDL文件生成Java文件的格式是固定,我们可以抛开AIDL文件直接写一个Binder出来,然而我觉得还是根据AIDL文件自动生成吧,但是这里我们需要知道AIDL文件并不是实现Binder的必需品,AIDL本质是系统为我们提供了一种快速实现Binder的工具,仅此而已.

接下来我们介绍Binder两个重要的方法linkToDeath和unlinkToDeath.我们知道,Binder运行在服务端进程,如果服务端进程由于某种原因终止,这个时候我们到服务端的Binder连接断裂(称之为Binder死亡),会导致我们远程调用失败,更加关键的是如果我们客户端不知道Binder连接已经断裂,那么客户端功能将会受影响,所以为了解决这个问题,Binder提供了两个配对的方法linkToDeath和unlinkToDeath,通过linkToDeath可以给Binder设置一个死亡代理,当Binder死亡时,我们就会收到通知,这个时候我们可以重新发起连接请求从而恢复连接,下面通过代码介绍.

首先声明一个DeathRecipient对象,实现其内部方法binderDied,当Binder死亡的时候,系统就会回调BinderDied方法,然后我们就可以移除之前绑定的binder代理并重新绑定远程服务

    private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {

            @Override
            public void binderDied() {
                Log.i("ppjun", "binderDied");

                if (mIBookManager == null)
                    return;
                mIBookManager.asBinder().unlinkToDeath(mDeathRecipient, 0);
                mIBookManager = null;

                Intent intent = new Intent("111");
                intent.setPackage("com.zly.www.ipcdemo");
                bindService(intent, new MyServiceConnection(), BIND_AUTO_CREATE);
            }
        };

其次在客户端绑定远程服务成功后,给binder设置死亡代理

    class MyServiceConnection implements ServiceConnection {

            @Override
            public void onServiceConnected(ComponentName name, IBinder binder) {
                mIBookManager = AbsBookManager.asInterface(binder);
                try {
                    binder.linkToDeath(mDeathRecipient, 0);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onServiceDisconnected(ComponentName name) {
            }
        }

其中linkToDeath第二个参数是标记为,我们给0就可以,当Binder死亡的时候我们就可以收到通知了,(然后我测试了一把是可以收到Binder死亡的通知,但是并不能拉起远程服务,所以其实没卵用),另外通过Binder的isBinderAlive也可以判断Binder是否死亡.

到此IPC基础介绍完全...累死宝宝,刚哥ipc这部分确实写的好所以有不少文字直接摘自刚哥的开发艺术了,再此表示感谢!!!
