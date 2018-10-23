# WebSocket安卓客户端实现详解(一)--连接建立与重连
>今年在公司第一个需求就是基于websocket写一个客户端消息中心,现在已经上线很久了在司机这种网络环境平均一天重连8次,自认为还是不错的.当时写的时候那个心酸啊,主要因为第一次写都不知道该从哪下手,没有方向.所以这里我将尽可能详细的跟大家分享出来.

本篇内容会比较多,先来段舞蹈热身下.

![](http://rocketzly.androider.top/icon_dance.gif)

我准备按如下顺序来讲解

1. 整体流程的一个概括了解大体思路.

2. 把大体流程细化,逐步去实现.

## 前言

**这里特别说明下因为WebSocket服务端是公司线上项目所以这里url和具体协议我全部抹去了,但我会尽力给大家讲明白并且demo我都是测试过,还望各位看官见谅**

我们先粗犷的讲下流程,掌握个大概的方向,然后在深入讲解细节的实现.这里先解答一个疑惑,为啥我们这要用WebSocket而不是Socket呢,因为WebSocket是一个应用层协议很多东西都规定好了我们直接按他的规定来用就好,而Socket是传输层和应用层的一个抽象层很多东西我们还得自己规定相对来说会比较麻烦,所以这里我们用的WebSocket.

既然WebSocket是一个应用层协议,我们肯定不可能自己去实现,所以第一步是需要找一个实现了该协议的框架,这里我用的[nv-websocket-client](https://github.com/TakahikoKawasaki/nv-websocket-client),api我就不介绍了,库中readme已经详细的介绍了,后面我就直接使用了.

关于通讯协议为了方便,这里我们使用的是json.

接下来我们先简单描述下我们将要做的事情

##  用户登录流程

![](http://rocketzly.androider.top/ws%20_summarize.png)

第一步用户输入账号密码登录成功后,我们将会通过websocket协议建立连接,当连接失败回调的时候我们尝试重连,直到连接成功,当然这个尝试重连的时间间隔我是根据重连失败次数按一定规则写的具体后面再说.

第二步当连接建立成功后,我们需要在后台通过长连接发送请求验证该用户的身份也就是上图的授权,既然前面用户登录都成功了一般情况下授权是不会失败的,所以这里对于授权失败并未处理,授权成功后我们开启心跳,并且发送同步数据请求到服务端获取还未收到的消息.

## 客户端发送请求流程

![](http://rocketzly.androider.top/WebSocket%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%AF%B7%E6%B1%82.png)

第一步将请求参数封装成请求对象,然后添加超时任务并且将该请求的回调添加到回调集合.

这里有点需要说明下,封装请求参数的时候这里额外添加了两个参数seqId和reqCount,这里我们是通过长连接请求当服务端响应的时候为了能够找到对应的回调,所以每个请求我们都需要传给服务端一个唯一标识来标识该请求,这里我用的seqId,请求成功后服务端再把seqId回传,我们再通过这个seqId作为key从回调集合中找到对应的回调.而reqCount的话主要针对请求超时的情况,如果请求超时,第二次请求的时候就把reqCount++在放入request中,我们约定同一个请求次数大于三次时候走http补偿通道,那么当request中的reqCount>3的时候我们就通过http发送该请求,然后根据响应回调对应结果.

第二步开始请求,成功或者失败的话通过seqId找到对应回调执行并从回调集合中移除该回调,然后取消超时任务.如果超时的话根据seqId拿到对应的回调并从回调集合中移除该回调,然后判断请求次数如果小于等于3次再次通过websocket尝试请求,如果大于3次通过http请求,根据请求成功失败情况执行对应回调.

## 服务端主动推送消息流程

![](http://rocketzly.androider.top/WebSocket%E6%9C%8D%E5%8A%A1%E7%AB%AF%E4%B8%BB%E5%8A%A8%E9%80%9A%E7%9F%A5.png)

先说明下这里服务端推送的消息仅仅是个事件,不携带具体消息.

第一步根据notify中事件类型找到对应的处理类,一般情况下这里需要同步对应数据.

第二步然后用eventbus通知对应的ui界面更新

第三步如果需要ack,发送ack请求

上面只是一个概括,对于心跳,重连,发送请求这里有不少细节需要注意的下一节我们将详细讲解

## 具体实现

理论说完了,接下来我们将一步步实现客户端代码.首先我们添加依赖

```gradle
    compile 'com.neovisionaries:nv-websocket-client:2.2'
```

然后创建一个单利的WsManager管理websocket供全局调用,

```java
public class WsManager {

    private static WsManager mInstance;

    private WsManager() {
    }

    public static WsManager getInstance(){
        if(mInstance == null){
            synchronized (WsManager.class){
                if(mInstance == null){
                    mInstance = new WsManager();
                }
            }
        }
        return mInstance;
    }
}
```

## 建立连接

然后添加建立连接代码,这里关于WebSocket协议的操作用的都是[nv-websocket-client](https://github.com/TakahikoKawasaki/nv-websocket-client),我也加上了详细的注释,实在不理解可以去读一遍readme文件.

```java
public class WsManager {
    private static WsManager mInstance;
    private final String TAG = this.getClass().getSimpleName();

    /**
     * WebSocket config
     */
    private static final int FRAME_QUEUE_SIZE = 5;
    private static final int CONNECT_TIMEOUT = 5000;
    private static final String DEF_TEST_URL = "测试服地址";//测试服默认地址
    private static final String DEF_RELEASE_URL = "正式服地址";//正式服默认地址
    private static final String DEF_URL = BuildConfig.DEBUG ? DEF_TEST_URL : DEF_RELEASE_URL;
    private String url;

    private WsStatus mStatus;
    private WebSocket ws;
    private WsListener mListener;

    private WsManager() {
    }

    public static WsManager getInstance(){
        if(mInstance == null){
            synchronized (WsManager.class){
                if(mInstance == null){
                    mInstance = new WsManager();
                }
            }
        }
        return mInstance;
    }

    public void init(){
        try {
          /**
           * configUrl其实是缓存在本地的连接地址
           * 这个缓存本地连接地址是app启动的时候通过http请求去服务端获取的,
           * 每次app启动的时候会拿当前时间与缓存时间比较,超过6小时就再次去服务端获取新的连接地址更新本地缓存
           */
            String configUrl = "";
            url = TextUtils.isEmpty(configUrl) ? DEF_URL : configUrl;
            ws = new WebSocketFactory().createSocket(url, CONNECT_TIMEOUT)
                .setFrameQueueSize(FRAME_QUEUE_SIZE)//设置帧队列最大值为5
                .setMissingCloseFrameAllowed(false)//设置不允许服务端关闭连接却未发送关闭帧
                .addListener(mListener = new WsListener())//添加回调监听
                .connectAsynchronously();//异步连接
            setStatus(WsStatus.CONNECTING);
            Logger.t(TAG).d("第一次连接");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    /**
     * 继承默认的监听空实现WebSocketAdapter,重写我们需要的方法
     * onTextMessage 收到文字信息
     * onConnected 连接成功
     * onConnectError 连接失败
     * onDisconnected 连接关闭
     */
    class WsListener extends WebSocketAdapter{
        @Override
        public void onTextMessage(WebSocket websocket, String text) throws Exception {
            super.onTextMessage(websocket, text);
            Logger.t(TAG).d(text);
        }


        @Override
        public void onConnected(WebSocket websocket, Map<String, List<String>> headers)
            throws Exception {
            super.onConnected(websocket, headers);
            Logger.t(TAG).d("连接成功");
            setStatus(WsStatus.CONNECT_SUCCESS);
        }


        @Override
        public void onConnectError(WebSocket websocket, WebSocketException exception)
            throws Exception {
            super.onConnectError(websocket, exception);
            Logger.t(TAG).d("连接错误");
            setStatus(WsStatus.CONNECT_FAIL);
        }


        @Override
        public void onDisconnected(WebSocket websocket, WebSocketFrame serverCloseFrame, WebSocketFrame clientCloseFrame, boolean closedByServer)
            throws Exception {
            super.onDisconnected(websocket, serverCloseFrame, clientCloseFrame, closedByServer);
            Logger.t(TAG).d("断开连接");
            setStatus(WsStatus.CONNECT_FAIL);
        }
    }

    private void setStatus(WsStatus status){
        this.mStatus = status;
    }

    private WsStatus getStatus(){
        return mStatus;
    }

    public void disconnect(){
        if(ws != null)
        ws.disconnect();
    }
}
```

```java
public enum WsStatus {
    CONNECT_SUCCESS,//连接成功
    CONNECT_FAIL,//连接失败
    CONNECTING;//正在连接
}
```

从注释我们可以知道,这里我们是app启动的时候通过http请求获取WebSocket连接地址,如果获取失败就走本地默认的url建立连接.并且内部自己维护了一个websocket状态后面发送请求和重连的时候会用上.

其实获取连接地址这个地方是可以优化的,就是app启动的时候先比较上次获取的时间如果大于6小时就通过http请求获取websocket的连接地址,这个地址应该是个列表,然后存入本地,连接的时候我们可以先ping下地址,选择耗时最短的地址接入.如果连不上我们在连耗时第二短的地址以此类推.但这里我们就以简单的方式做了.

至于建立连接代码在哪调用的话,我选择的是主界面`onCreate()`的时候,因为一般能进入主界面了,就代表用户已经登录成功.

```java
WsManager.getInstance().init();
```

断开连接的话在主界面`onDestroy()`的时候调用

```java
WsManager.getInstance().disconnect();
```

## 重连

建立连接有成功就有失败,对于失败情况我们需要重连,那么下面我们分别说明重连的时机,重连的策略和当前是否应该重连的判断.

对于重连的时机有如下几种情况我们需要尝试重连

1. 应用网络的切换.具体点就是可用网络状态的切换,比如4g切wifi连接会断开我们需要重连.

2. 应用回到前台的时候,判断如果连接断开我们需要重连,这个是尽量保持当应用再前台的时候连接的稳定.

3. 收到连接失败或者连接断开事件的时候,这个没什么好解释.

4. 心跳连续3次失败时候.当然这个连续失败3次是自己定义的,大伙可以根据自己app的情况定制.

等会我们先展示前三种情况,心跳失败这个在后面我们把客户端发送请求讲完再说.

上面把需要重连的情景说了,现在讲讲具体的重连策略.

这里我定义了一个最小重连时间间隔min和一个最大重连时间间隔max,当重连次数小于等于3次的时候都以最小重连时间间隔min去尝试重连,当重连次数大于3次的时候我们将重连地址替换成默认地址DEF_URL,将重连时间间隔按min*(重连次数-2)递增最大不不超过max.

还有最后一个当前是否应该重连的判断

1. 用户是否登录,可以通过本地是否有缓存的用户信息来判断.因为重连成功后我们需要将用户信息通过WebSocket发送到服务器进行身份验证所以这里必须登录成功.

2. 当前连接是否可用,这个通过[nv-websocket-client](https://github.com/TakahikoKawasaki/nv-websocket-client)库中的api判断`ws.isOpen()`.

3. 当前不是正在连接状态,这里我们根据自己维护的状态来判断`getStatus() != WsStatus.CONNECTING`.

4. 当前网络可用.

下面我们show code.跟之前相同的代码这里就省略了

```java
public class WsManager {

    .....省略部分跟之前代码一样.....

    /**
     * 继承默认的监听空实现WebSocketAdapter,重写我们需要的方法
     * onTextMessage 收到文字信息
     * onConnected 连接成功
     * onConnectError 连接失败
     * onDisconnected 连接关闭
     */
    class WsListener extends WebSocketAdapter {
        @Override
        public void onTextMessage(WebSocket websocket, String text) throws Exception {
            super.onTextMessage(websocket, text);
            Logger.t(TAG).d(text);
        }


        @Override
        public void onConnected(WebSocket websocket, Map<String, List<String>> headers)
            throws Exception {
            super.onConnected(websocket, headers);
            Logger.t(TAG).d("连接成功");
            setStatus(WsStatus.CONNECT_SUCCESS);
            cancelReconnect();//连接成功的时候取消重连,初始化连接次数
        }


        @Override
        public void onConnectError(WebSocket websocket, WebSocketException exception)
            throws Exception {
            super.onConnectError(websocket, exception);
            Logger.t(TAG).d("连接错误");
            setStatus(WsStatus.CONNECT_FAIL);
            reconnect();//连接错误的时候调用重连方法
        }


        @Override
        public void onDisconnected(WebSocket websocket, WebSocketFrame serverCloseFrame, WebSocketFrame clientCloseFrame, boolean closedByServer)
            throws Exception {
            super.onDisconnected(websocket, serverCloseFrame, clientCloseFrame, closedByServer);
            Logger.t(TAG).d("断开连接");
            setStatus(WsStatus.CONNECT_FAIL);
            reconnect();//连接断开的时候调用重连方法
        }
    }


    private void setStatus(WsStatus status) {
        this.mStatus = status;
    }


    private WsStatus getStatus() {
        return mStatus;
    }


    public void disconnect() {
        if (ws != null) {
            ws.disconnect();
        }
    }


    private Handler mHandler = new Handler();

    private int reconnectCount = 0;//重连次数
    private long minInterval = 3000;//重连最小时间间隔
    private long maxInterval = 60000;//重连最大时间间隔


    public void reconnect() {
        if (!isNetConnect()) {
            reconnectCount = 0;
            Logger.t(TAG).d("重连失败网络不可用");
            return;
        }

        //这里其实应该还有个用户是否登录了的判断 因为当连接成功后我们需要发送用户信息到服务端进行校验
        //由于我们这里是个demo所以省略了
        if (ws != null &&
            !ws.isOpen() &&//当前连接断开了
            getStatus() != WsStatus.CONNECTING) {//不是正在重连状态

            reconnectCount++;
            setStatus(WsStatus.CONNECTING);

            long reconnectTime = minInterval;
            if (reconnectCount > 3) {
                url = DEF_URL;
                long temp = minInterval * (reconnectCount - 2);
                reconnectTime = temp > maxInterval ? maxInterval : temp;
            }

            Logger.t(TAG).d("准备开始第%d次重连,重连间隔%d -- url:%s", reconnectCount, reconnectTime, url);
            mHandler.postDelayed(mReconnectTask, reconnectTime);
        }
    }


    private Runnable mReconnectTask = new Runnable() {

        @Override
        public void run() {
            try {
                ws = new WebSocketFactory().createSocket(url, CONNECT_TIMEOUT)
                    .setFrameQueueSize(FRAME_QUEUE_SIZE)//设置帧队列最大值为5
                    .setMissingCloseFrameAllowed(false)//设置不允许服务端关闭连接却未发送关闭帧
                    .addListener(mListener = new WsListener())//添加回调监听
                    .connectAsynchronously();//异步连接
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    };


    private void cancelReconnect() {
        reconnectCount = 0;
        mHandler.removeCallbacks(mReconnectTask);
    }


    private boolean isNetConnect() {
        ConnectivityManager connectivity = (ConnectivityManager) WsApplication.getContext()
            .getSystemService(Context.CONNECTIVITY_SERVICE);
        if (connectivity != null) {
            NetworkInfo info = connectivity.getActiveNetworkInfo();
            if (info != null && info.isConnected()) {
                // 当前网络是连接的
                if (info.getState() == NetworkInfo.State.CONNECTED) {
                    // 当前所连接的网络可用
                    return true;
                }
            }
        }
        return false;
    }
}
```

上面代码通过handler实现了一定时间间隔的重连,然后我们在WsListener监听中的`onConnectError()`和`onDisconnected()`调用了`reconnect()`实现重连,`onConnected()`中调用了`cancelReconnect()`取消重连并初始化重连次数.

所以当需要重连的时候我们调用`reconnect()`方法,如果失败`onConnectError()`和`onDisconnected()`回调会再次调用`reconnect()`实现重连,如果成功`onConnected()`中会调用`cancelReconnect()`取消重连并初始化重连次数.

并且这里我们已经实现了需要重连的情景3,收到连接失败或者连接断开事件的时候进行重连.

接下来我们实现情景1和2

1. 应用网络的切换.具体点就是可用网络状态的切换,比如4g切wifi连接会断开我们需要重连.

2. 应用回到前台的时候,判断如果连接断开我们需要重连,这个是尽量保持当应用再前台的时候连接的稳定.

对于可用网络的切换这里通过广播来监听实现重连

```java
public class NetStatusReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        if (ConnectivityManager.CONNECTIVITY_ACTION.equals(action)) {

            // 获取网络连接管理器
            ConnectivityManager connectivityManager
                = (ConnectivityManager) WsApplication.getContext()
                .getSystemService(Context.CONNECTIVITY_SERVICE);
            // 获取当前网络状态信息
            NetworkInfo info = connectivityManager.getActiveNetworkInfo();

            if (info != null && info.isAvailable()) {
                Logger.t("WsManager").d("监听到可用网络切换,调用重连方法");
                WsManager.getInstance().reconnect();//wify 4g切换重连websocket
            }

        }
    }
}
```

应用回到前台情况的重连.

通过`Application.ActivityLifecycleCallbacks`实现app前后台切换监听如下

```java
public class ForegroundCallbacks implements Application.ActivityLifecycleCallbacks {

    public static final long CHECK_DELAY = 600;
    public static final String TAG = ForegroundCallbacks.class.getName();
    private static ForegroundCallbacks instance;
    private boolean foreground = false, paused = true;
    private Handler handler = new Handler();
    private List<Listener> listeners = new CopyOnWriteArrayList<Listener>();
    private Runnable check;

    public static ForegroundCallbacks init(Application application) {
        if (instance == null) {
            instance = new ForegroundCallbacks();
            application.registerActivityLifecycleCallbacks(instance);
        }
        return instance;
    }

    public static ForegroundCallbacks get(Application application) {
        if (instance == null) {
            init(application);
        }
        return instance;
    }

    public static ForegroundCallbacks get(Context ctx) {
        if (instance == null) {
            Context appCtx = ctx.getApplicationContext();
            if (appCtx instanceof Application) {
                init((Application) appCtx);
            }
            throw new IllegalStateException(
                    "Foreground is not initialised and " +
                            "cannot obtain the Application object");
        }
        return instance;
    }

    public static ForegroundCallbacks get() {

        return instance;
    }

    public boolean isForeground() {
        return foreground;
    }

    public boolean isBackground() {
        return !foreground;
    }

    public void addListener(Listener listener) {
        listeners.add(listener);
    }

    public void removeListener(Listener listener) {
        listeners.remove(listener);
    }

    @Override
    public void onActivityResumed(Activity activity) {
        paused = false;
        boolean wasBackground = !foreground;
        foreground = true;
        if (check != null)
            handler.removeCallbacks(check);
        if (wasBackground) {

            for (Listener l : listeners) {
                try {
                    l.onBecameForeground();
                } catch (Exception exc) {

                }
            }
        } else {

        }
    }

    @Override
    public void onActivityPaused(Activity activity) {
        paused = true;

        if (check != null)
            handler.removeCallbacks(check);
        handler.postDelayed(check = new Runnable() {
            @Override
            public void run() {
                if (foreground && paused) {
                    foreground = false;
                    for (Listener l : listeners) {
                        try {
                            l.onBecameBackground();
                        } catch (Exception exc) {

                        }
                    }
                } else {

                }
            }
        }, CHECK_DELAY);
    }

    @Override
    public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
    }

    @Override
    public void onActivityStarted(Activity activity) {
    }

    @Override
    public void onActivityStopped(Activity activity) {
    }

    @Override
    public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
    }

    @Override
    public void onActivityDestroyed(Activity activity) {
    }

    public interface Listener {
        public void onBecameForeground();

        public void onBecameBackground();
    }
}
```

然后在application中初始化该监听,当应用回到前台的时候尝试重连

```java
public class WsApplication extends Application {


    @Override
    public void onCreate() {
        super.onCreate();
        initAppStatusListener();
    }

    private void initAppStatusListener() {
        ForegroundCallbacks.init(this).addListener(new ForegroundCallbacks.Listener() {
            @Override
            public void onBecameForeground() {
                Logger.t("WsManager").d("应用回到前台调用重连方法");
                WsManager.getInstance().reconnect();
            }

            @Override
            public void onBecameBackground() {

            }
        });
    }
}
```

到这里连接的建立和重连讲完了,还剩客户端发送请求和服务端主动通知消息.

本来我准备一篇把WebSocket客户端实现写完的,现在才一半就已经这么多了,索性分为几篇算了,下篇我们将介绍[ WebSocket安卓客户端实现详解(二)--客户端发送请求](http://blog.csdn.net/zly921112/article/details/76758424).

这里附上本篇的源码
[WebSocket安卓客户端实现详解(一)--连接建立与重连源码传送门](http://download.csdn.net/detail/zly921112/9866732)
