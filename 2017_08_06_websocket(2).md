# WebSocket安卓客户端实现详解(二)--客户端发送请求

本篇接着上一篇讲解WebSocket客户端发送请求和服务端主动通知消息,还没看过第一篇的小伙伴,这里附上第一篇链接[WebSocket安卓客户端实现详解(一)--连接建立与重连](http://blog.csdn.net/zly921112/article/details/72973054).

本篇依旧干货十足内容很多所以我们还是先热身下

![](http://rocketzly.androider.top/icon_%E6%83%B3%E5%AD%A6%E4%B9%A0.gif)

## 客户端发送请求

**为了方便我们协议使用json格式,当然你也可以替换成ProtoBuffer.**

这里先展示下发送请求的流程图,大体流程在第一篇已经讲过了这里不再赘述,直接搂细节.

![](http://rocketzly.androider.top/WebSocket%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%AF%B7%E6%B1%82.png)

既然是请求,那么得有一个请求协议.这里我们不直接展示协议,而是通过http请求,引导到websocket请求协议.

一般http请求我们会需要如下几个参数

1. url

2. 请求参数

3. 回调

而websocket是一个长连接,我们所有的请求都是通过这个连接发送的,也就是说我们没法像http那样通过一个url来区分不同请求,那么我们请求协议中就必须要有一个类似url的东西去区分请求,这里我用action字段来表示url,为了方便协议的扩展,我们在加一个参数req_event,由action和req_event共同组成url,下面是一个简单的例子.

```json
{
    "action": " login",
    "req_event": 0
}
```

由于请求是异步的所以当我们请求的时候需要把回调加入集合保存起来,等到服务端响应的时候我们需要从集合中找到对应回调调用对应的方法,那么为了能够找到对应回调我们需要给每个请求增加一个唯一标识,这里我用的seq_id字段表示,当请求的时候我们把客户端自己生成的seq_id传给服务端,然后当服务端响应的时候在回传给我们,这样我们就能通过seq_id作为key找到对应的回调,那么现在协议将变成下面这样.

```json
{
    "action": " login",
    "req_event": 0,
    "seq_id":0
}
```

在加上每个请求都有的请求参数,这里我用req字段表示,最终协议如下.

```json
{
    "action": " login",
    "req_event": 0,
    "seq_id":0,
    "req":{
      "aaa":"aaa",
      "bbb":0
    }
}
```

协议有了,那我们谈谈代码的实现,对于请求方法我们对外暴露如下参数.

1. Action

  这里我把action、req_event、响应实体统一用一个枚举类Action来存储,调用者只需根据不同请求传入对应Action即可.

2. Req

  请求参数

3. Callback

  回调

4. timeout(可选参数,默认给的10秒)

  请求超时时间.

show code

Action一个枚举类,把action、req_event、响应实体统一封装在一起,action和req_event用来组装url,响应实体在反序列化的时候会用到.调用者只需根据不同请求传入对应Action即可.

```java
public enum Action {
    LOGIN("login", 1, null);

    private String action;
    private int reqEvent;
    private Class respClazz;


    Action(String action, int reqEvent, Class respClazz) {
        this.action = action;
        this.reqEvent = reqEvent;
        this.respClazz = respClazz;
    }


    public String getAction() {
        return action;
    }


    public int getReqEvent() {
        return reqEvent;
    }


    public Class getRespClazz() {
        return respClazz;
    }

}
```

ui层回调

```java
public interface ICallback<T> {

    void onSuccess(T t);

    void onFail(String msg);

}
```

协议对应的请求实体类Request,这里还额外添加了一个没有参加序列化的参数reqCount来表示该请求的请求次数,在后面请求失败情况下对该请求的处理会用上.

```java
public class Request<T> {
    @SerializedName("action")
    private String action;

    @SerializedName("req_event")
    private int reqEvent;

    @SerializedName("seq_id")
    private long seqId;

    @SerializedName("req")
    private T req;

    private transient int reqCount;

    public Request(String action, int reqEvent, long seqId, T req, int reqCount) {
        this.action = action;
        this.reqEvent = reqEvent;
        this.seqId = seqId;
        this.req = req;
        this.reqCount = reqCount;
    }

    ....
    //这里还有各个参数对应get、set方法,为节省篇幅省略了

    public static class Builder<T> {
        private String action;
        private int reqEvent;
        private long seqId;
        private T req;
        private int reqCount;

        public Builder action(String action) {
            this.action = action;
            return this;
        }

        public Builder reqEvent(int reqEvent) {
            this.reqEvent = reqEvent;
            return this;
        }

        public Builder seqId(long seqId) {
            this.seqId = seqId;
            return this;
        }

        public Builder req(T req) {
            this.req = req;
            return this;
        }

        public Builder reqCount(int reqCount) {
            this.reqCount = reqCount;
            return this;
        }

        public Request build() {
            return new Request<T>(action, reqEvent, seqId, req, reqCount);
        }

    }
}
```

WsManager中发送请求代码

```java
public class WsManager{

  ....跟之前相同代码省略.....

  private static final int REQUEST_TIMEOUT = 10000;//请求超时时间
  private AtomicLong seqId = new AtomicLong(SystemClock.uptimeMillis());//每个请求的唯一标识

  public void sendReq(Action action, Object req, ICallback callback) {
      sendReq(action, req, callback, REQUEST_TIMEOUT);
  }


  public void sendReq(Action action, Object req, ICallback callback, long timeout) {
      sendReq(action, req, callback, timeout, 1);
  }


  /**
   * @param action Action
   * @param req 请求参数
   * @param callback 回调
   * @param timeout 超时时间
   * @param reqCount 请求次数
   */
  @SuppressWarnings("unchecked")
  private <T> void sendReq(Action action, T req, ICallback callback, long timeout, int reqCount) {
      if (!isNetConnect()) {
          callback.onFail("网络不可用");
          return;
      }

      Request request = new Request.Builder<T>()
          .action(action.getAction())
          .reqEvent(action.getReqEvent())
          .seqId(seqId.getAndIncrement())
          .reqCount(reqCount)
          .req(req)
          .build();

      Logger.t(TAG).d("send text : %s", new Gson().toJson(request));
      ws.sendText(new Gson().toJson(request));
  }

  ....跟之前相同代码省略.....

}
```

`sendReq(Action action, T req, ICallback callback, long timeout, int reqCount)`中有reqCount参数是因为在后面请求失败情况下会根据该请求进行请求的次数执行不同的策略.

## 超时和回调的处理


发送请求ok了接下来需要处理回调的问题了.虽然在方法`sendReq(Action action, T req, ICallback callback, long timeout, int reqCount)`中我们已经传入了ui层的回调ICallback,但这里还需要在封装一层回调处理一些通用逻辑,然后再调用ICallback对应方法.

需要处理的通用逻辑有如下三个.

1. 请求成功

  将服务端返回数据从子线程传到主线程,然后调用ui层回调.

2. 请求失败

  将失败信息从子线程传到主线程,然后调用ui层回调.

3. 请求超时

  该请求的reqCount <= 3再次通过websocket发送请求,reqCount > 3通过http通道发送请求,根据结果直接调用对应回调.


中间层回调定义如下

```java
public interface IWsCallback<T> {
    void onSuccess(T t);
    void onError(String msg, Request request, Action action);
    void onTimeout(Request request, Action action);
}
```

onSuccess与普通的成功回调一样,onError和onTimeout回调中有Request与Action是为了方便后续再次请求操作.

接下来showcode.

```java
  ....跟之前相同代码省略.....
  private final int SUCCESS_HANDLE = 0x01;
  private final int ERROR_HANDLE = 0x02;

  private Handler mHandler = new Handler(Looper.getMainLooper()) {
      @Override
      public void handleMessage(Message msg) {
          switch (msg.what) {
              case SUCCESS_HANDLE:
                  CallbackDataWrapper successWrapper = (CallbackDataWrapper) msg.obj;
                  successWrapper.getCallback().onSuccess(successWrapper.getData());
                  break;
              case ERROR_HANDLE:
                  CallbackDataWrapper errorWrapper = (CallbackDataWrapper) msg.obj;
                  errorWrapper.getCallback().onFail((String) errorWrapper.getData());
                  break;
          }
      }
  };

  private <T> void sendReq(Action action, T req, final ICallback callback, final long timeout, int reqCount) {
      if (!isNetConnect()) {
          callback.onFail("网络不可用");
          return;
      }

      Request request = new Request.Builder<T>()
          .action(action.getAction())
          .reqEvent(action.getReqEvent())
          .seqId(seqId.getAndIncrement())
          .reqCount(reqCount)
          .req(req)
          .build();

      IWsCallback temp = new IWsCallback() {

          @Override
          public void onSuccess(Object o) {
              mHandler.obtainMessage(SUCCESS_HANDLE, new CallbackDataWrapper(callback, o))
                  .sendToTarget();
          }


          @Override
          public void onError(String msg, Request request, Action action) {
              mHandler.obtainMessage(ERROR_HANDLE, new CallbackDataWrapper(callback, msg))
                  .sendToTarget();
          }


          @Override
          public void onTimeout(Request request, Action action) {
              timeoutHandle(request, action, callback, timeout);
          }
      };

      Logger.t(TAG).d("send text : %s", new Gson().toJson(request));
      ws.sendText(new Gson().toJson(request));
  }

  /**
   * 超时处理
   */
  private void timeoutHandle(Request request, Action action, ICallback callback, long timeout) {
      if (request.getReqCount() > 3) {
          Logger.t(TAG).d("(action:%s)连续3次请求超时 执行http请求", action.getAction());
          //走http请求
      } else {
          sendReq(action, request.getReq(), callback, timeout, request.getReqCount() + 1);
          Logger.t(TAG).d("(action:%s)发起第%d次请求", action.getAction(), request.getReqCount());
      }
  }

  ....跟之前相同代码省略.....
```

Handler中获取的Callback与Data包装类如下

```java
public class CallbackDataWrapper<T> {

    private ICallback<T> callback;
    private Object data;

    public CallbackDataWrapper(ICallback<T> callback, Object data) {
        this.callback = callback;
        this.data = data;
    }

    public ICallback<T> getCallback() {
        return callback;
    }


    public void setCallback(ICallback<T> callback) {
        this.callback = callback;
    }


    public Object getData() {
        return data;
    }


    public void setData(Object data) {
        this.data = data;
    }
}

```

从上面代码中可以看出对于需要处理的逻辑1和2在回调中通过CallbackDataWrapper包装callback和data发送到主线程handler在调用对应的回调处理.

需要处理的逻辑3则在`timeoutHandle()`方法中实现,具体http请求这里我没实现,大家只需要调用自己项目封装的http请求api即可,当然为了方便这里建议还是以我们前面定义的websocket协议的形式执行http请求,这样我们就不需要重新组装请求参数了.

超时后的处理有了,接下来我们实现添加超时任务代码.

```java
....跟之前相同代码省略.....

private ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
private Map<Long, CallbackWrapper> callbacks = new HashMap<>();

@SuppressWarnings("unchecked")
private <T> void sendReq(Action action, T req, final ICallback callback, final long timeout, int reqCount) {
    if (!isNetConnect()) {
        callback.onFail("网络不可用");
        return;
    }

    Request request = new Request.Builder<T>()
        .action(action.getAction())
        .reqEvent(action.getReqEvent())
        .seqId(seqId.getAndIncrement())
        .reqCount(reqCount)
        .req(req)
        .build();

    ScheduledFuture timeoutTask = enqueueTimeout(request.getSeqId(), timeout);//添加超时任务

    IWsCallback tempCallback = new IWsCallback() {

        @Override
        public void onSuccess(Object o) {
            mHandler.obtainMessage(SUCCESS_HANDLE, new CallbackDataWrapper(callback, o))
                .sendToTarget();
        }


        @Override
        public void onError(String msg, Request request, Action action) {
            mHandler.obtainMessage(ERROR_HANDLE, new CallbackDataWrapper(callback, msg))
                .sendToTarget();
        }


        @Override
        public void onTimeout(Request request, Action action) {
            timeoutHandle(request, action, callback, timeout);
        }
    };

    callbacks.put(request.getSeqId(),
        new CallbackWrapper(tempCallback, timeoutTask, action, request));

    Logger.t(TAG).d("send text : %s", new Gson().toJson(request));
    ws.sendText(new Gson().toJson(request));
}

/**
 * 添加超时任务
 */
private ScheduledFuture enqueueTimeout(final long seqId, long timeout) {
    return executor.schedule(new Runnable() {
        @Override
        public void run() {
            CallbackWrapper wrapper = callbacks.remove(seqId);
            if (wrapper != null) {
                Logger.t(TAG).d("(action:%s)第%d次请求超时", wrapper.getAction().getAction(), wrapper.getRequest().getReqCount());
                wrapper.getTempCallback().onTimeout(wrapper.getRequest(), wrapper.getAction());
            }
        }
    }, timeout, TimeUnit.MILLISECONDS);
}

....跟之前相同代码省略.....
```

这里我们用的`ScheduledThreadPoolExecutor`来处理超时任务,通过`ScheduledThreadPoolExecutor.schedule()`方法间隔一定时长执行超时的处理.当然为了能进行超时的处理我们需要将callback保存起来,又由于超时的时候我们需要进行再次请求所以不仅需要callback还需要request与action,至于执行超时任务返回的ScheduledFuture这个我们会在服务端响应的时候取消超时任务用上这边我提前添加了.最终通过CallbackWrapper将他们包装了起来.

callback包装类

```java

public class CallbackWrapper {

    private final IWsCallback tempCallback;
    private final ScheduledFuture timeoutTask;
    private final Action action;
    private final Request request;


    public CallbackWrapper(IWsCallback tempCallback, ScheduledFuture timeoutTask, Action action, Request request) {
        this.tempCallback = tempCallback;
        this.timeoutTask = timeoutTask;
        this.action = action;
        this.request = request;
    }


    public IWsCallback getTempCallback() {
        return tempCallback;
    }


    public ScheduledFuture getTimeoutTask() {
        return timeoutTask;
    }


    public Action getAction() {
        return action;
    }


    public Request getRequest() {
        return request;
    }
}

```

接下来是请求的最后一步了服务端响应的处理.这里我们还是先从协议入手.

首先说明下对于长连接无论是我们请求的响应还是服务端的主动通知都是通过同一个回调方法回调给客户端我们没法区分,所以服务端给我们数据中需要带一个标志来区分普通请求的响应与服务端主动通知.这里我们用resp_event来表示,当resp_event为10的时候是请求的响应,为20的时候是服务端的主动通知.

对于请求的响应我们需要通过seq_id找到对应的回调,对于主动通知我们需要添加一个action来区分这个通知执行什么操作,在加上响应数据体resp协议就定制完成了.当然seq_id应该在客户端请求的响应的时候才有,action应该在服务端主动通知的时候才有.这里为了解析的方便最外层通用协议如下.

```json
{
    "resp_event": 10,
    "action": "",
    "seq_id": 11111111,
    "resp": {}
}
```

对于我们客户端主动请求的响应数据可以按照http请求的格式来定义code,msg,data那么最终数据结构如下

```json
{
    "resp_event": 10,
    "action": "",
    "seq_id": 11111111,
    "resp": {
      "code":0,
      "msg":"ok",
      "data":{

      }
    }
}
```

对于服务端主动的推送不需要code和msg那么数据结构改为如下

```json
{
    "resp_event": 20,
    "action": "",
    "seq_id": 11111111,
    "resp": {

      }
    }
}
```

可以看出最外层json数据结构一样,从resp开始不同.这里对于客户端主动请求的响应的情况我已经封装好了Codec工具类去解析,服务端主动推送的话后面再说。

最外层bean如下

```java
public class Response {

    @SerializedName("resp_event")
    private int respEvent;

    @SerializedName("seq_id")
    private String seqId;

    private String action;
    private String resp;

    //省略get set方法
}

```

第二层bean如下

```java
public class ChildResponse {

    private int code;
    private String msg;
    private String data;

    public boolean isOK(){
        return code == 0;
    }

    //省略get set方法
}

```

对应的解析工具类

```java
public class Codec {

    public static Response decoder(String text) {
        Response response = new Response();
        JsonParser parser = new JsonParser();
        JsonElement element = parser.parse(text);
        if (element.isJsonObject()) {
            JsonObject obj = (JsonObject) element;
            response.setRespEvent(decoderInt(obj, "resp_event"));
            response.setAction(decoderStr(obj, "action"));
            response.setSeqId(decoderStr(obj, "seq_id"));
            response.setResp(decoderStr(obj, "resp"));
            return response;
        }
        return response;
    }

    private static int decoderInt(JsonObject obj, String name) {
        int result = -1;
        JsonElement element = obj.get(name);
        if (null != element) {
            try {
                result = element.getAsInt();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return result;
    }

    private static String decoderStr(JsonObject obj, String name) {
        String result = "";
        try {
            JsonElement element = obj.get(name);
            if (null != element && element.isJsonPrimitive()) {
                result = element.getAsString();
            } else if (null != element && element.isJsonObject()) {
                result = element.getAsJsonObject().toString();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
    }

    public static ChildResponse decoderChildResp(String jsonStr) {
        ChildResponse childResponse = new ChildResponse();
        JsonParser parser = new JsonParser();
        JsonElement element = parser.parse(jsonStr);
        if (element.isJsonObject()) {
            JsonObject jsonObject = (JsonObject) element;
            childResponse.setCode(decoderInt(jsonObject, "code"));
            childResponse.setMsg(decoderStr(jsonObject, "msg"));
            childResponse.setData(decoderStr(jsonObject, "data"));
        }
        return childResponse;
    }

}

```

接下来就是响应的处理了

```java
....跟之前相同代码省略.....

class WsListener extends WebSocketAdapter {
    @Override
    public void onTextMessage(WebSocket websocket, String text) throws Exception {
        super.onTextMessage(websocket, text);
        Logger.t(TAG).d("receiverMsg:%s", text);

        Response response = Codec.decoder(text);//解析出第一层bean
        if (response.getRespEvent() == 10) {//响应
            CallbackWrapper wrapper = callbacks.remove(Long.parseLong(response.getSeqId()));//找到对应callback
            if (wrapper == null) {
                Logger.t(TAG).d("(action:%s) not found callback", response.getAction());
                return;
            }

            try {
                wrapper.getTimeoutTask().cancel(true);//取消超时任务
                ChildResponse childResponse = Codec.decoderChildResp(response.getResp());//解析第二层bean
                if (childResponse.isOK()) {

                    Object o = new Gson().fromJson(childResponse.getData(),
                        wrapper.getAction().getRespClazz());

                    wrapper.getTempCallback().onSuccess(o);
                } else {
                    wrapper.getTempCallback()
                        .onError(ErrorCode.BUSINESS_EXCEPTION.getMsg(), wrapper.getRequest(),
                            wrapper.getAction());
                }
            } catch (JsonSyntaxException e) {
                e.printStackTrace();
                wrapper.getTempCallback()
                    .onError(ErrorCode.PARSE_EXCEPTION.getMsg(), wrapper.getRequest(),
                        wrapper.getAction());
            }

        } else if (response.getRespEvent() == 20) {//通知

        }
    }
}

....跟之前相同代码省略.....
```

先解析出第一层bean然后根据respEvent判断响应类型,当respEvent为10的时候通过seqId找对应的callback如果找到了则取消超时任务,解析第二层bean判断code是否为成功,失败调用失败回调,成功用gson解析对应的bean然后调用成功回调.

错误信息的封装ErrorCode如下

```java
public enum ErrorCode {
    BUSINESS_EXCEPTION("业务异常"),
    PARSE_EXCEPTION("数据格式异常"),
    DISCONNECT_EXCEPTION("连接断开");

    private String msg;

    ErrorCode(String msg) {
        this.msg = msg;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

## 授权和心跳

发送请求封装好了接下来我们可以开始授权和心跳了,这里我们先回顾下用户登录流程.

![](http://rocketzly.androider.top/ws%20_summarize.png)

可以看到我们连接成功后进行授权,授权就是发送一个携带用户信息的请求然后服务端通过这个请求验证用户信息,验证成功后服务端就知道了当前长连接属于哪个用户,由于我们是先通过http请求进行的登录然后在通过websocket进行的授权,既然http登录能成功所以正常情况下通过websocket进行授权不会有问题,所以这里错误回调没有进行处理.

这里我们模拟这个过程,首先是定义请求的action,在action中添加请求对应的action,reqEvent,respClazz这个比较简单我就省略了,然后在连接成功回调进行授权请求`doAuth()`

```java
public class WsManager {

    ....跟之前相同代码省略.....

    private void doAuth() {
        sendReq(Action.LOGIN, null, new ICallback() {
            @Override
            public void onSuccess(Object o) {
                Logger.t(TAG).d("授权成功");
                setStatus(WsStatus.AUTH_SUCCESS);
            }


            @Override
            public void onFail(String msg) {

            }
        });
    }

    /**
     * 继承默认的监听空实现WebSocketAdapter,重写我们需要的方法
     * onTextMessage 收到文字信息
     * onConnected 连接成功
     * onConnectError 连接失败
     * onDisconnected 连接关闭
     */
    class WsListener extends WebSocketAdapter {

        @Override
        public void onConnected(WebSocket websocket, Map<String, List<String>> headers)
            throws Exception {
            super.onConnected(websocket, headers);
            Logger.t(TAG).d("连接成功");
            setStatus(WsStatus.CONNECT_SUCCESS);
            cancelReconnect();//连接成功的时候取消重连,初始化连接次数
            doAuth();
        }

    }

....跟之前相同代码省略.....
}

```

授权成功后接下来是心跳和同步数据,本质跟授权一样也是发送请求.

这里单独说下心跳,心跳其实就是每隔一段时间我们请求服务端然后服务端有响应我们就认为这条连接是稳定的,对于websocket其实已经定义了心跳的格式ping和pong,为了方便这里我就直接使用我们自己定义的协议发送的心跳请求,本质上跟ping和pong是一样的只是协议不同罢了.对于心跳重连策略我这里做的比较简单按一个固定时间执行心跳请求,当心跳连续失败3次就进行重连.

首先还是定义action.这里只是模拟所以respClazz都是null

```java
public enum Action {
    LOGIN("login", 1, null),
    HEARTBEAT("heartbeat", 1, null),
    SYNC("sync", 1, null);

    private String action;
    private int reqEvent;
    private Class respClazz;


    Action(String action, int reqEvent, Class respClazz) {
        this.action = action;
        this.reqEvent = reqEvent;
        this.respClazz = respClazz;
    }


    public String getAction() {
        return action;
    }


    public int getReqEvent() {
        return reqEvent;
    }


    public Class getRespClazz() {
        return respClazz;
    }

}
```

心跳和同步

```java
public class WsManager {
  ....跟之前相同代码省略.....

    private static final long HEARTBEAT_INTERVAL = 30000;//心跳间隔
    private Handler mHandler = new Handler(Looper.getMainLooper()) {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case SUCCESS_HANDLE:
                    CallbackDataWrapper successWrapper = (CallbackDataWrapper) msg.obj;
                    successWrapper.getCallback().onSuccess(successWrapper.getData());
                    break;
                case ERROR_HANDLE:
                    CallbackDataWrapper errorWrapper = (CallbackDataWrapper) msg.obj;
                    errorWrapper.getCallback().onFail((String) errorWrapper.getData());
                    break;
            }
        }
    };

    private void doAuth() {
        sendReq(Action.LOGIN, null, new ICallback() {
            @Override
            public void onSuccess(Object o) {
                Logger.t(TAG).d("授权成功");
                setStatus(WsStatus.AUTH_SUCCESS);
                startHeartbeat();
                delaySyncData();
            }


            @Override
            public void onFail(String msg) {

            }
        });
    }

    //同步数据
    private void delaySyncData() {
        mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                sendReq(Action.SYNC, null, new ICallback() {
                    @Override
                    public void onSuccess(Object o) {

                    }


                    @Override
                    public void onFail(String msg) {

                    }
                });
            }
        }, 300);
    }


    private void startHeartbeat() {
        mHandler.postDelayed(heartbeatTask, HEARTBEAT_INTERVAL);
    }


    private void cancelHeartbeat() {
        heartbeatFailCount = 0;
        mHandler.removeCallbacks(heartbeatTask);
    }


    private int heartbeatFailCount = 0;
    private Runnable heartbeatTask = new Runnable() {
        @Override
        public void run() {
            sendReq(Action.HEARTBEAT, null, new ICallback() {
                @Override
                public void onSuccess(Object o) {
                    heartbeatFailCount = 0;
                }


                @Override
                public void onFail(String msg) {
                    heartbeatFailCount++;
                    if (heartbeatFailCount >= 3) {
                        reconnect();
                    }
                }
            });

            mHandler.postDelayed(this, HEARTBEAT_INTERVAL);
        }
    };

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
            cancelHeartbeat();//取消心跳

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

    ....跟之前相同代码省略.....
}

```

授权成功后开始心跳和同步数据,心跳连续失败三次开始重连,在重连时候会取消心跳.当重连成功的时候在连接成功回调会再次进行授权然后授权成功后会再次开启心跳就形成了一个循环.

感觉篇幅又有些长了..为了避免大伙疲劳将服务端主动通知放第三篇[WebSocket安卓客户端实现详解(三)--服务端主动通知](http://blog.csdn.net/zly921112/article/details/76767876)好了,并且WebSocket整个模块我觉得我写的最好的就是服务端主动通知这部分还请各位看官一定要捧场啊.

源码下载[WebSocket安卓客户端实现详解(二)--客户端发送请求](http://download.csdn.net/download/zly921112/9922714)
