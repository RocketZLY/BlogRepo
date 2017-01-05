>最近有空就在看IPC,正好路过AIDL这里,发现基本忘干净了...正好借此机会回顾并记录下来,那么接下来我就先撸为敬了.

## **AIDL用来做什么**

AIDL（Android 接口定义语言）与您可能使用过的其他 IDL 类似。 您可以利用它定义客户端与服务使用进程间通信 (IPC) 进行相互通信时都认可的编程接口。 在 Android 上，一个进程通常无法访问另一个进程的内存。 尽管如此，**进程需要将其对象分解成操作系统能够识别的原语，并将对象编组成跨越边界的对象**。 编写执行这一编组操作的代码是一项繁琐的工作，因此 Android 会使用 AIDL 来处理。

上面是文档中的描述,通俗的说法:AIDL的作用是让你可以在自己的APP里绑定一个其他APP的service并可以拿到它暴露给你的方法获取数据，这样你的APP可以和其他APP交互。

## **AIDL使用**

上面已经说过AIDL是用来将对象分解成操作系统能够识别的原语，并将对象编组成跨越边界的对象,那么这里就按AIDL中可以使用的数据类型来分别说明如何使用.

### **基本类型**

因为AIDL是两个APP交互啦，所以当然要两个APP啦，我们在第一个工程目录右键

![](http://of1ktyksz.bkt.clouddn.com/aidl_new_inter.png)

输入名字AS就帮我们新建了AIDL文件了

    // IMyName.aidl
    package com.zly.www.test1;

    // Declare any non-default types here with import statements

    interface IMyName {
        /**
         * Demonstrates some basic types that you can use as parameters
         * and return values in AIDL.
         */
        void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
                double aDouble, String aString);
    }

这里basicTypes这个方法其实没啥用可以删掉,但从他注释里面可以看出,AIDL默认支持int,long,boolean,float,double,String基本类型的数据.因为这里是要跨进程通讯的，所以不是随便你自己定义的一个类型就可以在AIDL使用的,自定义类型我们在后面会讲.我们在AIDL文件中定义一个我们要提供给第二个APP使用的接口.

那么我们把这个类稍微修改下

    // IMyName.aidl
    package com.zly.www.test1;

    // Declare any non-default types here with import statements

    interface IMyName {

        String getName();

    }

然后点下sycn project让AS帮我们生成下aidl代码.

![](http://of1ktyksz.bkt.clouddn.com/aidl_generate_code)

接下来我们新建一个service并且在清单注册,然后在service中新建一个内部类MyBinder继承刚刚写的AIDL接口IMyName里的Stub类并实现方法,在onBind()返回该实例,你可能会问我没在IMyName里写Stub类呀,其实这里Stub是AS根据刚刚AIDL文件生成的代码中的.

    public class MyService extends Service {

        private MyBinder mBinder = new MyBinder();

        @Nullable
        @Override
        public IBinder onBind(Intent intent) {
            return mBinder;
        }

        class MyBinder extends IMyName.Stub{

            @Override
            public String getName() throws RemoteException {
                return "帅也是错吗";
            }
        }
    }

接下来新建一个项目,将前面写的AIDL文件拷贝过来,**要注意的是包名必须完全一致.**

![](http://of1ktyksz.bkt.clouddn.com/aidl_copy_interface.png)

syncProject一下项目,然后在MainActivity中绑定service

    public class MainActivity extends AppCompatActivity {

        private IMyName iMyName;
        private ServiceConnection serviceConnection;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);

            Intent intent = new Intent("aaaa");
            intent.setPackage("com.zly.www.test1");
            bindService(intent, serviceConnection = new ServiceConnection() {
                @Override
                public void onServiceConnected(ComponentName name, IBinder service) {
                    iMyName = IMyName.Stub.asInterface(service);
                }

                @Override
                public void onServiceDisconnected(ComponentName name) {
                    iMyName = null;
                }
            }, BIND_AUTO_CREATE);

        }

        public void get(View v) {
            try {
                Toast.makeText(this, iMyName.getName(), Toast.LENGTH_SHORT).show();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        protected void onDestroy() {
            super.onDestroy();
            unbindService(serviceConnection);
        }
    }

这边我们通过隐式意图来绑定service，在onServiceConnected方法中通过IMyName.Stub.asInterface(service)获取IMyName对象，然后在get()中调用IMyName.getName()。(这里有点需要注意Android 5.0出后，其中有个特性就是Service Intent must be explitict,也就是说从Lollipop开始,service服务必须采用显示方式启动,所以这里intent我指定了package)

![](http://of1ktyksz.bkt.clouddn.com/aidl_base_preview.gif)

## **自定义类型**
如果要在AIDL中使用自定义类型的数据,第一自定义类型要实现Parcelable接口,第二需要写一个ADIL声明文件.

下面创建一个Student类实现Parcelable接口

    public class Student implements Parcelable {

        public Student(String name) {
            this.name = name;
        }

        private String name;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        protected Student(Parcel in) {
            name = in.readString();
        }

        @Override
        public void writeToParcel(Parcel dest, int flags) {
            dest.writeString(name);
        }

        @Override
        public int describeContents() {
            return 0;
        }

        public static final Creator<Student> CREATOR = new Creator<Student>() {
            @Override
            public Student createFromParcel(Parcel in) {
                return new Student(in);
            }

            @Override
            public Student[] newArray(int size) {
                return new Student[size];
            }
        };

    }

接下来新建一个aidl文件，名称为我们自定义类型的名称，这边是Student.aidl。在Student.aidl申明我们的自定义类型和它的完整包名，注意这边parcelable是小写的，不是Parcelable接口，一个自定类型需要一个这样同名的AIDL文件来声明。

    package com.zly.www.test1;
    parcelable Student;

然后在我们的aidl接口中导入自定义数据类型

![](http://of1ktyksz.bkt.clouddn.com/aidl_import_custom_type.png)

syncProjcet,然后在service中实现接口方法

    class MyBinder extends IMyName.Stub{

            @Override
            public String getName() throws RemoteException {
                return "帅也是错吗";
            }

            @Override
            public Student getStudent() throws RemoteException {
                return new Student("朱利源");
            }
        }

然后把AIDL文件与自定义类型文件拷到第二个项目,**注意这里还是包名一致**

![](http://of1ktyksz.bkt.clouddn.com/aidl_client.png)

接下来就可以在该activity中使用自定义类型的数据了

    public class MainActivity extends AppCompatActivity {
        //.....
        public void get(View v) {
            try {
                Toast.makeText(this, iMyName.getStudent().getName(), Toast.LENGTH_SHORT).show();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

    }

效果如下

![](http://of1ktyksz.bkt.clouddn.com/aidl_custom_preview.gif)
