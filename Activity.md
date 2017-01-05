>本系列主要是读书笔记,知识点百分之90来自 <<安卓开发艺术探索>>和<<安卓群英传>>,还有百分之10是自己的扩展与理解.欢迎吐槽

生命周期
-----------------
**一.正常情况下生命周期如图**
![这里写图片描述](http://img.blog.csdn.net/20160911182533967)
正常生命周期 开起activity调用onCreate() onStart() onResume(),按下返回键 onPause() onStop() onDestroy()

启动activity onCreate() onStart() onResume()
  切换到桌面或者打开一个新的activity  onPause() onStop() 返回该activity onRestart() onStart() onResume()
  如果打开新的activity主题设置为透明 则 onPause() 返回该activity onResume()

<font color=#FF4500>这里唯一需要注意的是,当前activity onPause()方法执行后 才会执行新的activity的onCreate(),也就是说onPause()不能进行耗时操作,不然会影响下个activity显示速度</font>

![这里写图片描述](http://img.blog.csdn.net/20160911183227623)
</br>
</br>
**二.异常情况生命周期**
系统配置发生改变 或者 系统内存不足情况下发生

**1.系统配置发生改变**
默认情况下,如果activity不做特殊处理,那么系统配置发生改变后,activity就会被销毁并重新创建
![这里写图片描述](http://img.blog.csdn.net/20160911183451573)
系统配置发生改变后,activity会被销毁,即onPause(),onStop(),onDestroy()会执行,由于是异常情况下终止的,系统会调用onSaveInstanceState()来保存当前状态,该方法调用在onStop()之前,<font color=#FF4500>(要纠正下的是该方法不止是只有异常情况下才会调用,正常情况比如activityA打开activityB也会执行onSaveInstanceState()方法这里有日志为证)</font>
![这里写图片描述](http://img.blog.csdn.net/20160911183906922)
虽然跟我们平时听说的有不太一样...但日志如此,实践是检验真理的唯一标准.
当activity重建后,系统会调用onRestoreInstanceState()并把销毁时onSaveInstanceState()所保存的Bundle对象作为参数传递给onRestoreInstanceState ()和onCreate()(两者区别是:onRestoreInstanceState()被调用其参数Bundle saveInstanceState一定有值,而onCreate()如果正常启动的话,其参数Bundle saveInstanceState 为null,所以必须要额外的非空判断),从调用顺序上来说onRestoreInstanceState()在onStart()之后并且只有异常情况下才会调用.

同时,在onSaveInstanceState()与onRestoreInstanceState()方法中,系统自动为我们做了一定的恢复工作,例如异常情况下页面重启的时候,系统会默认为我们恢复当前activity视图结构(文本框输入数据,listview滚动位置等),具体针对每个特定的view系统能为我们恢复那些数据,可以查看view源码,每个view都有
onSaveInstanceState()与onRestoreInstanceState()方法,看下他的实现,就知道系统能自动为每个View恢复哪些数据了.

**2.资源内存不足导致低优先级的activity被杀死**
activity优先级从高到低可以分为
前台activity->可见但非前台activity(比如activity中弹出了一个对话框,导致activity可见但是不可操作)->后台activity,因为内存不足activity重启的生命周期和之前1的并无差异

</br>
通过上面分析,我们知道系统配置发生改变后,activity会被重新创建,那么有没有办法不重新创建呢?答案是有的,系统配置中有很多内容,如果当某个配置发生改变,我们不想创建activity可以给他指定configChanges属性,具体可以看下面表格
|VALUE|DESCRIPTION|
| ------------- | ------------- |
|"mcc"|国际移动用户识别码所属国家代号是改变了-----  sim被侦测到了，去更新mcc    mcc是移动用户所属国家代号|
|"mnc"|国际移动用户识别码的移动网号码是改变了------ sim被侦测到了，去更新mnc    MNC是移动网号码，最多由两位数字组成，用于识别移动用户所归属的移动通信网|
|"locale"|地址改变了-----用户选择了一个新的语言会显示出来|
|"touchscreen"|触摸屏是改变了------通常是不会发生的|
|"keyboard"|键盘发生了改变----例如用户用了外部的键盘|
|"keyboardHidden"|键盘的可用性发生了改变|
|"navigation"|导航发生了变化-----通常也不会发生|
|"screenLayout"|屏幕的显示发生了变化------不同的显示被激活|
|"fontScale"|字体比例发生了变化----选择了不同的全局字体|
|"uiMode"|用户的模式发生了变化|
|"orientation"|屏幕方向改变了|
|"screenSize"|屏幕大小改变了|
|"smallestScreenSize"|屏幕的物理大小改变了，如：连接到一个外部的屏幕上|
上面属性只有一部分,具体以实际情况为准,下面来一个例子,指定configChanges属性为orientation|screenSize,activity横竖屏切换后,不会重新创建

```
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:name=".base.AppApplication"
    android:supportsRtl="true"
    android:theme="@style/AppTheme">
    <activity android:name=".MainActivity"
        android:configChanges="orientation|screenSize">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>

    <activity android:name=".TakePicture" />
</application>
```
横竖屏切换后,activity并未重启,而是调用了onConfigurationChanged()![这里写图片描述](http://img.blog.csdn.net/20160911185143194)                   	                                                                                                                                                                                      

---------------------------------
启动模式
--------
目前有四种启动模式standard,singleTop,singleTask和singleInstance

**一.standard**
标准模式,也是系统默认的模式,每次启动一个activity都会创建一个新的实例,不管这个实例是否存在,生命周期就是标准的生命周期,在这种模式下谁启动了这个activity,那么这个activity就运行在启动它的那个activity所在栈中, 最终会形成的特色栈况为： ABCDEF…….或者ABCDDCEFAB……..

**二.singleTop**
栈顶复用模式,如果新activity位于栈顶,此时启动该activity,该activity并不会被重新创建,同时他的onNewInstance方法会被调用, 所以不会出现ABCDD,而只会有ABCD ,生命周期如下
![这里写图片描述](http://img.blog.csdn.net/20160911185503772)
如果activity实例已经存在但是不位于栈顶,那么新activity仍会创建

**三.singleTask**
栈内复用模式,这是一种单实例模式,只要activity在一个栈中存在,那么多次启动activity都不会重新创建实例,系统会把该activity切换到栈顶并调用其onNewIntent()方法 (栈内所有在他之上的activity全部出栈 )

具体点,当一个具有singleTask模式的Activity请求启动后,比如ActivityA,系统首先会寻找是否存在ActivityA想要的任务栈(ps:这个想要的任务栈根据AndroidManifest.xml文件中TaskAffinity属性的值来决定,如果当前有值并且跟系统默认的任务栈包名不相同则会创建新的任务栈再将activity压入栈中,详情后面会说到),如果不存在,就重新创建一个任务栈,然后创建一个A的实例后把A放入栈中,如果存在A所需的任务栈,这时候要看A是否在栈中有实例存在,如果有实例存在,那么系统会把A调到栈顶调用其onNewInstance方法并把在他之上的activity全部出栈,如果实例不存在,就创建A实例并把A压入栈中.

这里验证下TaskAffinity属性
1.启动模式SingleTask的Activity不设置TaskAffinity(等价于想要的任务栈为默认的名字为包名的任务栈),是否创建新的任务栈在压入Activity

```
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:name=".base.AppApplication"
    android:supportsRtl="true"
    android:theme="@style/AppTheme">
    <activity android:name=".MainActivity"
        android:configChanges="orientation|screenSize"
       >
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>

    <activity android:name=".TwoActivity"
        android:launchMode="singleTask"
        />

</application>
```
清单就给TwoActivity设置singleTask模式,在每个Activity onCreat()方法打印TaskId

```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    KLog.i("MainActivity:onCreate");
    int taskId = getTaskId();
    KLog.i("MainActivity:taskId"+taskId);

}
```
日志如下taskId未改变并未创建新的任务栈,因为TaskAffinity未设置.
![这里写图片描述](http://img.blog.csdn.net/20160911185847215)

2.设置TaskAffinity,看是否创建新的任务栈
清单如下

```
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:name=".base.AppApplication"
    android:supportsRtl="true"
    android:theme="@style/AppTheme">
    <activity android:name=".MainActivity"
        android:configChanges="orientation|screenSize"
       >
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>

    <activity android:name=".TwoActivity"
        android:launchMode="singleTask"
        android:taskAffinity="cn.yy"
        />

</application>
```
其他没有变化,日志如下
![这里写图片描述](http://img.blog.csdn.net/20160911190035312)
日志中taskId改变了,确实创建了新的任务栈.结论正确

**四.singleInstance**
单实例模式,加强版的singleTask模式,具有此种模式的activity只能单独的位于一个任务栈中,比如第一次启动singleInstance模式的ActivityA,系统会先创建一个新的任务栈,然后A独自在这个新的任务栈中,再次启动该activity由于栈内复用的特性均不会创建activity,除非这个任务站被系统销毁了

这里验证一下 singleInstance 模式的activity只能单独的位于一个任务栈中
MainActivity 启动singleInstance模式的TwoActivity, Two在启动一个正常模式的ThreeActivity,比较Two与Three任务栈是否相同
```
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:name=".base.AppApplication"
    android:supportsRtl="true"
    android:theme="@style/AppTheme">
    <activity android:name=".MainActivity">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>

    <activity android:name=".TwoActivity"
        android:launchMode="singleInstance"
        />

    <activity android:name=".ThreeActivity"/>
</application>
```
每个activity,onCreate()方法如下

```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    KLog.i("MainActivity onCreate");
    int taskId = getTaskId();
    KLog.i("MainActivity taskId"+taskId);
}
```
日志如下
![这里写图片描述](http://img.blog.csdn.net/20160911190325378)
可以看到ThreeActivity与TwoActivity是不同的任务栈,证明TwoActivity确实是独占一个任务栈.

<font color=#FF4500>需要注意的是activity任务栈,这得从一个参数说起TaskAffinity,这个参数标示了一个activity所需任务栈的名字,默认情况下,所有activity所需的任务栈的名字都为应用的包名,当然,我们我可以为每个activity都指定独立的TaskAffinity属性,这个值必须和包名不相同,否则相当于没指定,另外需要注意的是TaskAffinity需要SingleTask或者allowTaskReparenting属性配合使用,否则没有意义.另外任务栈分为前台任务栈和后台任务栈,用户可以通过切换将后台任务栈再次调到前台.</font>

**五.Activity的Flags**
这里只介绍几个常用的,
**1.FLAG_ACTIVITY_NEW_TASK(默认)**
默认的跳转类型,它会重新创建一个新的Activity，不过有这种情况，比如说Task1中有A,B,C三个Activity,此时在C中启动D的话，如果在AndroidManifest.xml文件中给D添加了TaskAffinity的值和默认的任务栈(包名)不一样的话，则会创建一个新的任务栈名字为TaskAffinity的值,然后压入这个Activity。如果是默认的或者指定的 TaskAffinity 和Task一样的话，就和标准模式一样了启动一个新的Activity.

**2.FLAG_ACTIVITY_SINGLE_TOP**
这个FLAG就相当于启动模式中的singletop，例如:原来栈中结构是A B C D，在D中启动D，栈中的情况还是A,B,C,D。

**3.FLAG_ACTIVITY_CLEAR_TOP**
如果在ABCD的堆栈状态下，以该标识启动B，则会销毁CD，且B也是重新创建的（与singleTask有区别），如果配合FLAG_ACTIVITY_SINGLE_TOP，则就会成为singleTask的模式

</br>
</br>

----------------------------------
IntentFilter匹配规则
----------------
activity启动分为两种,隐式调用和显示调用,

**1.显示调用**
就像启动Activity，我们常常就是显式的调用，那何为显式调用呢？

```
Intent itent = new Intent();
itent.setClass(Activity_A.this, Activity_B.class);
startActivity(itent);
```

哦，这就是显式调用。之说以叫做显式调用，我们为Intent清楚的指出了被启动组件的信息（这里就是Activity_B）,当调用了startActivity(itent)后，我们就只会很明确的知道，这次的任务是启动Activity_B,而没有其它的过程。

**2.隐式调用**
看了显式调用，应该猜都猜得到了，隐式调用就是没有明确的指出组件信息。而是通过Filter去过滤出需要的组件。
```
Intent intent = new Intent();
intent.setAction(Intent.ACTION_BATTERY_LOW);
intent.addCategory(Intent.CATEGORY_APP_EMAIL);
intent.setDataAndType(Uri.EMPTY, "video/mpeg");
startActivity(intent);
```
这里就是一个隐式的调用，可以看到我为Intent设置了三个属性Action、Category、Data。
然后startActivity(intent)就会根据我们设置的这三个属性去筛选合适的组件来打开，也就是因为这样，所以有时候，当我们APP来分享一个东西的时候，会有很多组件（比如QQ、微信、微博...）来供我们选择，因为他们都满足Filter条件。

**2.1action**
action是一个字符串,匹配规则是Intent中必须有一个action且必须能够和过滤规则中的action的字符串完全一样,需要注意的是action区分大小写

**2.2category**
category是一个字符串,category 要求Intent中可以没有category，但是你一旦有category，不管几个，它必须是IntentFilter中定义了的category。

<font color=#FF4500>这里我们说Intent中可以没有category,其实不然，只是在我们启动组件（eg：startActivity( )）的时候，默认给我们的Intent给加了一个category("android.intent.category.DEFAULT" ).</font>

哦，我们知道了这里，那么匹配就和action差不多了，就是我们的Intent中的category必须在IntentFilter中存在。这里得注意，Intent中都会包括默认的category,并且如果你想隐式启动某个组件，那么就得在IntentFilter中添加android.intent.category.DEFAULT这个category才行哟。

**2.3data**
先了解下data结构
```
<data android:scheme="string"
          android:host="string"
          android:port="string"
          android:path="string"
          android:pathPattern="string"
          android:pathPrefix="string"
          android:mimeType="string" />
```
data由两部分组成,mimeType和URI,mimeType指媒体类型,比如image/jpeg,video/*等,可以便是图片,音频等不同媒体格式,而URI包含的就比较多了
```
<scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]
```
这个给个例子就比较好理解了

```
http://www.baidu.com:80/serch/info
```
Scheme:URI的模式,比如http,file,content等
Host:URI的主机名,比如www.baidu.com
port:端口号,比如80,仅当URI中指定了scheme和host参数的时候port才有意义
path,pathPattern和pathPrefix:这三个参数标示路径信息,path标示完整路径信息,pathPattern也标示完成路径信息,但是它里面可以包含通配符*,来代替0个或者多个任意字符,pathPrefix标示路径前缀信息

data匹配规则,
data数据能够完全匹配过滤规则中某一个data
举个栗子

```
	 <intent-filter >  
	                 <action android:name="com.intenttest.OTHER"/>  
	                 <category android:name="android.intent.category.DEFAULT"/>  
	                 <data android:scheme="test" android:host="www.google.com" android:port="80"/>  
	                 <data android:scheme="test2"/>  
	                 <data android:mimeType="text/*" />  
	             </intent-filter>  
```
匹配代码如下

```
	Intent intent = new Intent();  
	intent.setAction("com.intenttest.OTHER ");  
	Uri data = Uri.parse("test://www.google.com:80");  
	intent.setDataAndType(data, "text/*");  
	startActivity(intent);  

```


```
* 如果设置了多个data，只要匹配一个就可以启动这个activity。

* 如果设置了 <data android:scheme="test" android:host="www.google.com" android:port="80"/>，必须完全匹配 Uri data = Uri.parse("test://www.google.com:80");才能启动。

* 如果 <data android:scheme="test" android:host="www.google.com"/>，那么 Uri data = Uri.parse("test://www.google.com:80")，Uri data = Uri.parse("test://www.google.com:88")， Uri data = Uri.parse("test://www.google.com")都可以匹配。

* 如果只设置了 <data android:scheme="test"/>，那么 Uri data = Uri.parse("test://")就可以匹配，后面也可以加其他参数。

* 如果设置了mimeType，那么必须使用 intent.setDataAndType(data, "text/*");启动activity。
```


最后,我们隐式方式启动activity的时候最好加上判断避免没有匹配的activity的错误,
利用Intent的resolveActivity方法,如果找不到匹配的组件就会返回null

另外action和category中,有一类比较重要

```
<action android:name="android.intent.action.MAIN" />

<category android:name="android.intent.category.LAUNCHER" />
```
这两者共同作用是用来标明这是一个activity入口,并且会出现在系统的应用列表中,少了任何一个都没有意义,也无法出现在系统的应用列表中,
