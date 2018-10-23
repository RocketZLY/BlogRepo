# WebSocket安卓客户端实现详解(三)--服务端主动通知


本篇依旧是接着上一篇继续扩展,还没看过之前博客的小伙伴,这里附上前几篇地址

[ WebSocket安卓客户端实现详解(一)--连接建立与重连](http://blog.csdn.net/zly921112/article/details/72973054)

[WebSocket安卓客户端实现详解(二)--客户端发送请求](http://blog.csdn.net/zly921112/article/details/76758424)

终于是最后一篇啦,有点激动\ ( ≧▽≦ ) /啦啦啦,

![](http://rocketzly.androider.top/icon_xiong.gif)


## 服务端主动通知

热身完毕,我们先回顾下第一篇中讲到的服务端主动通知的流程

![](http://rocketzly.androider.top/WebSocket%E6%9C%8D%E5%8A%A1%E7%AB%AF%E4%B8%BB%E5%8A%A8%E9%80%9A%E7%9F%A5.png)

1. 根据notify中事件类型找到对应的处理类,处理对应逻辑.

2. 然后用eventbus通知对应的ui界面更新.

3. 如果需要ack,发送ack请求.

在回顾下第二篇中服务端主动通知协议的格式

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

我们根据resp_event为20判断这次响应是服务端主动通知,然后通过action找到对应处理类,然后把resp中数据解析成对应的bean传入对应处理类执行对应业务逻辑.

show code

```java
public class WsManager {

  ....跟之前相同代码省略.....

  class WsListener extends WebSocketAdapter {
      @Override
      public void onTextMessage(WebSocket websocket, String text) throws Exception {
          super.onTextMessage(websocket, text);
          Logger.t(TAG).d("receiverMsg:%s", text);

          Response response = Codec.decoder(text);//解析出第一层bean
          if (response.getRespEvent() == 10) {//响应
              CallbackWrapper wrapper = callbacks.remove(
                      Long.parseLong(response.getSeqId()));//找到对应callback
              if (wrapper == null) {
                  Logger.t(TAG).d("(action:%s) not found callback", response.getAction());
                  return;
              }

              try {
                  wrapper.getTimeoutTask().cancel(true);//取消超时任务
                  ChildResponse childResponse = Codec.decoderChildResp(
                          response.getResp());//解析第二层bean
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
              NotifyListenerManager.getInstance().fire(response);
          }
      }

  }

  ....跟之前相同代码省略.....

}
```

我们先解析出第一层bean然后根据resp_event为20执行NotifyListenerManager通知管理类对外暴露的fire()方法.

```java
public class NotifyListenerManager {
    private final String TAG = this.getClass().getSimpleName();
    private volatile static NotifyListenerManager manager;
    private Map<String, INotifyListener> map = new HashMap<>();

    private NotifyListenerManager() {
        regist();
    }

    public static NotifyListenerManager getInstance() {
        if (manager == null) {
            synchronized (NotifyListenerManager.class) {
                if (manager == null) {
                    manager = new NotifyListenerManager();
                }
            }
        }
        return manager;
    }

    private void regist() {
        map.put("notifyAnnounceMsg", new AnnounceMsgListener());
    }

    public void fire(Response response) {
        String action = response.getAction();
        String resp = response.getResp();
        INotifyListener listener = map.get(action);
        if (listener == null) {
            Logger.t(TAG).d("no found notify listener");
            return;
        }

        NotifyClass notifyClass = listener.getClass().getAnnotation(NotifyClass.class);
        Class<?> clazz = notifyClass.value();
        Object result = null;
        try {
            result = new Gson().fromJson(resp, clazz);
        } catch (JsonSyntaxException e) {
            e.printStackTrace();
        }
        Logger.t(TAG).d(result);
        listener.fire(result);
    }


}

```

NotifyListenerManager是一个单例的类,在第一次创建的时候在构造方法中执行了regist方法,这是一个变种的观察者模式对于添加观察者这个过程我们直接在regist方法中写好了,如果增加了新的业务逻辑我们只需要在regist方法中put新添加的action与对应处理类.对外暴露的fire方法根据传入的responsse中action找到对应的处理类,拿到处理类对应的注解标记的class,将服务端返回的resp解析成对应的bean丢到对应处理类执行对应逻辑.

```java
//抽象接口
public interface INotifyListener<T> {
    void fire(T t);
}

//标记注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface NotifyClass {

    Class<?> value();

}

//具体逻辑对应的处理子类
@NotifyClass(AnnounceMsgNotify.class)
public class AnnounceMsgListener implements INotifyListener<AnnounceMsgNotify> {

    @Override
    public void fire(AnnounceMsgNotify announceMsgNotify) {
        //这里处理具体的逻辑
    }
}

//对应数据bean
public class AnnounceMsgNotify {
    @SerializedName("msg_version")
    private String msgVersion;

    public String getMsgVersion() {
        return msgVersion;
    }

    public void setMsgVersion(String msgVersion) {
        this.msgVersion = msgVersion;
    }

}
```

如果新增业务逻辑我们只需要实现新的业务逻辑类,然后在NotifyListenerManager的regist方法中put新增的action与listener映射关系,对外只需要调用`NotifyListenerManager.getInstance().fire(response)`即可,实现了解耦.

到此websocket介绍完啦....鼓掌鼓掌鼓掌.

## 总结

对于websocket使用我已经尽我所能最详细的讲解了一遍,但是也避免不了有所疏漏和错误还望各位小伙伴指出.

然后虽然写了三篇但是还有几个点说的不够详细,这里我一一列举感兴趣的小伙伴可以自己看看.

1. 获取连接地址与选择连接地址的策略

2. 重连的策略

3. 心跳的策略

4. 进程保活

最后感谢各位小伙伴捧场能把三篇都看完的绝对是真爱啊...

源码下载[WebSocket安卓客户端实现详解(三)--服务端主动通知](http://download.csdn.net/detail/zly921112/9922705)
