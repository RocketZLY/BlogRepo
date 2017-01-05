>最近在搞项目6.0权限问题,正好借着这股劲把之前一直不太理解的compileSdkVersion, minSdkVersion 和 targetSdkVersion这三个属性看了下,看完后不禁的发出感慨原来就是这样啊...

本文出自于对以下两篇文章的整理总结
http://chinagdg.org/2016/01/picking-your-compilesdkversion-minsdkversion-targetsdkversion/
http://www.race604.com/android-targetsdkversion/

这里先做个简单的介绍,后面详细的说明

- minSdkVersion:应用可以运行的最低要求

- compileSdkVersion:控制可以使用哪个版本的api

- targetSdkVersion:应用的兼容模式

**minSdkVersion**
----------------------
----------------------------------------------------------
见名之意,app运行所需的最低sdk版本.低于minSdkVersion的手机将无法安装.

在开发时 minSdkVersion 也起到一个重要角色：lint 默认会在项目中运行，它在你使用了高于 minSdkVersion  的 API 时会警告你，帮你避免调用不存在的 API 的运行时问题(比如 minSdkVersion 9但你使用了 sdkVersion 10 才有的API就会警告 )。

如果只在较高版本的系统上才使用某些 API，通常使用运行时检查系统版本的方式解决。
比如这样
```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP){
            //do something
}

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT && Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP){
            //do something
}
```
<br/>

**compileSdkVersion**
--------------------------
--------------------------------------------------
compileSdkVersion 告诉 Gradle 用哪个 Android SDK 版本编译你的应用。使用任何新添加的 API 就需要使用对应版本的 Android SDK。

修改 compileSdkVersion 不会改变运行时的行为。当你修改了 compileSdkVersion 的时候，可能会出现新的编译警告、编译错误，但新的 compileSdkVersion 不会被包含到 APK 中：它纯粹只是在编译的时候使用。

注意，如果使用 Support Library ，那么使用最新发布的 Support Library 就需要使用最新的 SDK 编译。例如，要使用 23.1.1 版本的 Support Library ，compileSdkVersion 就必需至少是 23 。
<br/>

**targetSdkVersion**
----------------------------
-----------------------------------------------------------
三个属性中最难懂的就是这个了**targetSdkVersion 是 Android 提供向前兼容的主要依据**，在应用的 targetSdkVersion 没有更新之前系统不会应用最新的行为变化。

具体点就是随着 Android 系统的升级，某个系统的 API 或者模块的行为可能会发生改变，但是为了保证老 APK 的行为还是和以前兼容。只要 APK 的 targetSdkVersion 不变，即使这个 APK 安装在新 Android 系统上，其行为还是保持老的系统上的行为，这样就保证了系统对老应用的前向兼容性。

举个栗子
在 Android 4.4 (API 19）以后，AlarmManager 的 set() 和 setRepeat() 这两个 API 的行为发生了变化。在 Android 4.4 以前，这两个 API 设置的都是精确的时间，系统能保证在 API 设置的时间点上唤醒 Alarm。因为省电原因 Android 4.4 这两个 API 设置的唤醒时间，系统都对待成不精确的时间，系统只能保证在你设置的时间点之后某个时间唤醒。

这时，虽然 API 没有任何变化，但是实际上 API 的行为却发生了变化，如果老的 APK 中使用了此 API，并且在应用中的行为非常依赖 AlarmManager 在精确的时间唤醒，例如闹钟应用。如果 Android 系统不能保证兼容，老的 APK 安装在新的系统上，就会出现问题。

Android 系统是怎么保证这种兼容性的呢？这时候 targetSdkVersion 就起作用了。APK 在调用系统 AlarmManager 的 set() 或者 setRepeat() 的时候，系统首先会查一下调用的 APK 的 targetSdkVersion 信息，如果小于 19，就还是按照老的行为，即精确设置唤醒时间，否者执行新的行为。

我们来看一下 Android 4.4 上 AlarmManger 的一部分源代码：

```
private final boolean mAlwaysExact;  
AlarmManager(IAlarmManager service, Context ctx) {  
    mService = service;

    final int sdkVersion = ctx.getApplicationInfo().targetSdkVersion;
    mAlwaysExact = (sdkVersion < Build.VERSION_CODES.KITKAT);
}
```
看到这里，首选获取应用的 targetSdkVersion，判断是否是小于 Build.VERSION_CODES.KITKAT (即 API Level 19)，来设置 mAlwaysExact 变量，表示是否使用精确时间模式。

```
public static final long WINDOW_EXACT = 0;  
public static final long WINDOW_HEURISTIC = -1;

private long legacyExactLength() {  
    return (mAlwaysExact ? WINDOW_EXACT : WINDOW_HEURISTIC);
}

public void set(int type, long triggerAtMillis, PendingIntent operation) {  
    setImpl(type, triggerAtMillis, legacyExactLength(), 0, operation, null);
}
```
这里看到，直接影响到 set() 方法给 setImpl() 传入不同的参数，从而影响到了 set() 的执行行为。具体的实现在 AlarmManagerService.java，这里就不往下深究了。

看到这里，发现其实 Android 的 targetSdkVersion 并没有什么特别的，系统使用它也非常直接，甚至很“粗糙”。仅仅是用过下面的 API 来获取 targetSdkVersion，来判断是否执行哪种行为：

```
getApplicationInfo().targetSdkVersion;
```
我们可以猜测到，如果 Android 系统升级，发生这种兼容行为的变化时，一般都会在原来的保存新旧两种逻辑，并通过 if-else 方法来判断执行哪种逻辑。果然，在源码中搜索，我们会发现不少类似 getApplicationInfo().targetSdkVersion < Buid.XXXX 这样的代码，相对于浩瀚的 Android 源码量来说，这些还是相对较少了。其实原则上，这种会导致兼容性问题的修改还是越少越好，所以每次发布新的 Android 版本的时候，Android 开发者网站都会列出做了哪些改变，在这里，开发者需要特别注意。

所以当我们修改了 APK 的 targetSdkVersion 后行为会发生变化，也就必须做完整的测试了。
<br/>

综合来看三个属性的关系是
minSdkVersion <= targetSdkVersion <= compileSdkVersion

理想上，在稳定状态下三者的关系应该更像这样：
minSdkVersion (lowest possible) <= targetSdkVersion == compileSdkVersion (latest SDK)
用较低的 minSdkVersion 来覆盖最大的人群，用最新的 SDK 设置 target 和 compile 来获得新版本最好的效果。
