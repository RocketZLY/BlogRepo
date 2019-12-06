# RetryAndFollowUpInterceptor源码解析

> 本文基于okhttp3.10.0

近段时间准备陆陆续续把okhttp拦截器分析一遍，既然是本系列开篇那先来回顾下拦截器是干嘛的，然后在开始今天的主题。

## 1. 拦截器简介

先简单看下拦截器是干嘛的，okhttp默认给我们提供了5个拦截器

- RetryAndFollowUpInterceptor
- BridgeInterceptor 
- CacheInterceptor
- ConnectInterceptor
- CallServerInterceptor

当我们通过okhttp发送请求的时候是将请求封装为一个Request对象，然后包装成一个即将请求的Call对象，无论是异步或者同步请求最终都是通过RealCall#getResponseWithInterceptorChain()方法获取请求结果，那么我们看下方法体

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

可以看到是将拦截器组合起来创建一个RealInterceptorChain对象并将第五个参数赋值为0，**而第五个参数是赋值给了RealInterceptorChain中的index成员变量**，然后通过chain#proceed(originalRequest)获取请求结果。

```java
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    ...
    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);//index+1链条的下一个节点
    Interceptor interceptor = interceptors.get(index);//拿到第一个拦截器
    Response response = interceptor.intercept(next);
    ...
    return response;
  }
```

第一次进来的时候index为0所以拿到的是第一个拦截器，而interceptor#intercept()传入的RealInterceptorChain的index为index+1即为1，所以当拦截器中调用chain#proceed(originalRequest)则会调到下个拦截器形成一个递归关系，完成整个拦截器链的调用。

```java
//拦截器伪代码
public class Interceptor {
  Response intercept(Chain chain) throws IOException{
    Request request = chain.request();
    //对Request做操作
    Response response = chain.process()
    //对Response做操作
  }
}
```

![](http://rocketzly.androider.top/okhttp%E6%8B%A6%E6%88%AA%E5%99%A8%E8%B0%83%E7%94%A8%E9%93%BE)

那么可以简单的理解拦截器就是处理请求的，输入一个请求返回一个响应，按照http协议约定的代码实现。

## 2. RetryAndFollowUpInterceptor做了啥

1. 创建StreamAllocation它会在 ConnectInterceptor 中真正被使用到，主要就是用于获取连接服务端的 Connection 和用于进行跟服务端进行数据传输的输入输出流 HttpStream，具体的操作后面用到再说
2. 出错重试
3. 继续请求（主要是重定向）

## 2.1 出错重试

```java
  @Override public Response intercept(Chain chain) throws IOException {
    while (true) {
      if (canceled) {//请求取消了释放连接
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;//默认释放连接
      try {
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;//请求成功暂不释放连接
      } catch (RouteException e) {
        //出现异常判断能不能进行重试
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;//可以重试不释放连接
        continue;//通过continue继续执行while循环重试
      } catch (IOException e) {
        //出现异常判断能不能进行重试
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;//可以重试不释放连接
        continue;//通过continue继续执行while循环重试
      } finally {
        //不能重试则释放资源
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }
    }
  }

```

请求取消了释放连接，请求成功暂不释放连接，请求抛出RouteException和IOException通过recover()判断能不能进行重试，如果可以也暂不释放连接，并通过continue继续进行while循环重新请求。

```java
  private boolean recover(IOException e, StreamAllocation streamAllocation,
      boolean requestSendStarted, Request userRequest) {
    streamAllocation.streamFailed(e);

    // The application layer has forbidden retries.
    if (!client.retryOnConnectionFailure()) return false;//应用层可以配置拒绝重试

    // We can't send the request body again.
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;//如果请求已经发送并且body实现了UnrepeatableRequestBody则不重试

    // This exception is fatal.
    if (!isRecoverable(e, requestSendStarted)) return false;//请求发生致命错误

    // No more routes to attempt.
    if (!streamAllocation.hasMoreRoutes()) return false;//没有更多的路线

    // For failure recovery, use the same route selector with a new connection.
    return true;
  }
```

第一个if是调用client#retryOnConnectionFailure()判断能不能进行重试

```java
  public boolean retryOnConnectionFailure() {
    return retryOnConnectionFailure;
  }
```

而它实际是读取的OkHttpClient的retryOnConnectionFailure属性默认为true支持重试

第二个if是判断如果请求已经发送，并且body实现了UnrepeatableRequestBody则不再重试，UnrepeatableRequestBody在这里只是做一个标记的作用

第三个if判断是不是致命异常，如果是则不再重试

```java
  private boolean isRecoverable(IOException e, boolean requestSendStarted) {
    // If there was a protocol problem, don't recover.
    if (e instanceof ProtocolException) {//出现协议错误
      return false;
    }

    // If there was an interruption don't recover, but if there was a timeout connecting to a route
    // we should try the next route (if there is one).
    if (e instanceof InterruptedIOException) {//中断异常
      return e instanceof SocketTimeoutException && !requestSendStarted;
    }

    // Look for known client-side or negotiation errors that are unlikely to be fixed by trying
    // again with a different route.
    if (e instanceof SSLHandshakeException) {//ssl握手异常
      // If the problem was a CertificateException from the X509TrustManager,
      // do not retry.
      if (e.getCause() instanceof CertificateException) {
        return false;
      }
    }
    if (e instanceof SSLPeerUnverifiedException) {//验证异常
      // e.g. a certificate pinning error.
      return false;
    }

    // An example of one we might want to retry with a different route is a problem connecting to a
    // proxy and would manifest as a standard IOException. Unless it is one we know we should not
    // retry, we return true and try a new route.
    return true;
  }

```

第四个if如果没有更多的线路则不进行重试，其他情况则进行重试

## 2.2 继续请求（主要是重定向）

```java
  @Override public Response intercept(Chain chain) throws IOException {
    int followUpCount = 0;//继续请求次数
    Response priorResponse = null;//上次的响应
    while (true) {
      //...省略请求重试代码

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {//如果有前一个响应则附加到responses中
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Request followUp = followUpRequest(response, streamAllocation.route());//判断需不需要继续请求

      if (followUp == null) {//不需要继续请求
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;//直接返回响应
      }
			//下面则是需要继续请求
      closeQuietly(response.body());//去掉body

      if (++followUpCount > MAX_FOLLOW_UPS) {//如果继续请求次数大于20次报错终止
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {//如果body实现了UnrepeatableRequestBody接口不进行继续请求
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      if (!sameConnection(response, followUp.url())) {//不能重用连接
        streamAllocation.release();//释放旧的连接
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;//记录继续请求的request
      priorResponse = response;//上次的response
    }
  }
```

代码注释已经很全了，直接看到对于继续请求的判断followUpRequest()

```java
  private Request followUpRequest(Response userResponse, Route route) throws IOException {
    if (userResponse == null) throw new IllegalStateException();
    int responseCode = userResponse.code();//拿到http状态码

    final String method = userResponse.request().method();//请求方法
    switch (responseCode) {
      case HTTP_PROXY_AUTH://407代理服务器验证
        Proxy selectedProxy = route != null
            ? route.proxy()
            : client.proxy();
        if (selectedProxy.type() != Proxy.Type.HTTP) {
          throw new ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy");
        }
        return client.proxyAuthenticator().authenticate(route, userResponse);//返回一个附带验证信息的Request
      case HTTP_UNAUTHORIZED://401和407类似，只不过401是服务器验证失败
        return client.authenticator().authenticate(route, userResponse);//返回一个附带验证信息的Request

      case HTTP_PERM_REDIRECT://308
      case HTTP_TEMP_REDIRECT://307
        // "If the 307 or 308 status code is received in response to a request other than GET
        // or HEAD, the user agent MUST NOT automatically redirect the request"
        if (!method.equals("GET") && !method.equals("HEAD")) {//307 308如果不是get或者head的话则不在进行重定向
          return null;
        }
        // fall-through
      case HTTP_MULT_CHOICE://300
      case HTTP_MOVED_PERM://301
      case HTTP_MOVED_TEMP://302
      case HTTP_SEE_OTHER://303
        // Does the client allow redirects?
        if (!client.followRedirects()) return null;//客户端是否允许重定向 默认是允许

        String location = userResponse.header("Location");//获取响应头中重定向地址
        if (location == null) return null;
        HttpUrl url = userResponse.request().url().resolve(location);//解析成HttpUrl对象

        // Don't follow redirects to unsupported protocols.
        if (url == null) return null;

        // If configured, don't follow redirects between SSL and non-SSL.
        boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
        if (!sameScheme && !client.followSslRedirects()) return null;//是否允许http到https重定向

        // Most redirects don't include a request body.
        Request.Builder requestBuilder = userResponse.request().newBuilder();
        if (HttpMethod.permitsRequestBody(method)) {//如果请求方法不是get或者head
          final boolean maintainBody = HttpMethod.redirectsWithBody(method);//请求方法为PROPFIND需要有body
          if (HttpMethod.redirectsToGet(method)) {//如果请求方法不是PROPFIND重定向为get请求并去除请求体
            requestBuilder.method("GET", null);
          } else {//请求方法是PROPFIND，加入请求体并保持原有请求方法
            RequestBody requestBody = maintainBody ? userResponse.request().body() : null;
            requestBuilder.method(method, requestBody);
          }
          if (!maintainBody) {//如果没有请求体则移除对应头部字段
            requestBuilder.removeHeader("Transfer-Encoding");
            requestBuilder.removeHeader("Content-Length");
            requestBuilder.removeHeader("Content-Type");
          }
        }

        // When redirecting across hosts, drop all authentication headers. This
        // is potentially annoying to the application layer since they have no
        // way to retain them.
        if (!sameConnection(userResponse, url)) {//不是同一个连接移除Authorization字段
          requestBuilder.removeHeader("Authorization");
        }

        return requestBuilder.url(url).build();

      case HTTP_CLIENT_TIMEOUT://408 进行重试
        // 408's are rare in practice, but some servers like HAProxy use this response code. The
        // spec says that we may repeat the request without modifications. Modern browsers also
        // repeat the request (even non-idempotent ones.)
        if (!client.retryOnConnectionFailure()) {//判断是否允许重试
          // The application layer has directed us not to retry the request.
          return null;
        }

        if (userResponse.request().body() instanceof UnrepeatableRequestBody) {
          return null;
        }

        if (userResponse.priorResponse() != null
            && userResponse.priorResponse().code() == HTTP_CLIENT_TIMEOUT) {//发生两次408则不在重试
          // We attempted to retry and got another timeout. Give up.
          return null;
        }

        if (retryAfter(userResponse, 0) > 0) {//response中Retry-After有值则不再重试
          return null;
        }

        return userResponse.request();

      case HTTP_UNAVAILABLE://503 默认不重试
        if (userResponse.priorResponse() != null
            && userResponse.priorResponse().code() == HTTP_UNAVAILABLE) {//出现两次503则不再重试
          // We attempted to retry and got another timeout. Give up.
          return null;
        }

        if (retryAfter(userResponse, Integer.MAX_VALUE) == 0) {//response中包含Retry-After并且值为0进行重试
          // specifically received an instruction to retry without delay
          return userResponse.request();
        }

        return null;

      default:
        return null;
    }
  }
```

这里就是对http响应码做区分看是否需要再次请求，先来科普下http响应码

![](http://rocketzly.androider.top/http%E7%8A%B6%E6%80%81%E7%A0%81%E5%88%86%E7%B1%BB.png)

- 401 & 407 身份验证
- 300 & 301 & 302 & 304 & 307 & 308 重定向
- 408 客户端超时不常用
- 503 服务器不可用

返回码为401&407的话，获取带验证信息的Request重新请求，408这个不常用默认是会重试的，503为服务不可用默认不重试，接下来是重头戏3xx重定向。

```java
      case HTTP_PERM_REDIRECT://308
      case HTTP_TEMP_REDIRECT://307
        if (!method.equals("GET") && !method.equals("HEAD")) {//307 308如果不是get或者head的话则不在进行重定向
          return null;
        }
      case HTTP_MULT_CHOICE://300
      case HTTP_MOVED_PERM://301
      case HTTP_MOVED_TEMP://302
      case HTTP_SEE_OTHER://303
        if (!client.followRedirects()) return null;//客户端是否允许重定向 默认是允许

        String location = userResponse.header("Location");//获取响应头中重定向地址
        if (location == null) return null;
        HttpUrl url = userResponse.request().url().resolve(location);//解析成HttpUrl对象

        if (url == null) return null;

        boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
        if (!sameScheme && !client.followSslRedirects()) return null;//是否允许http到https重定向

        Request.Builder requestBuilder = userResponse.request().newBuilder();
        if (HttpMethod.permitsRequestBody(method)) {//如果请求方法不是get或者head
          final boolean maintainBody = HttpMethod.redirectsWithBody(method);//请求方法为PROPFIND需要有body
          if (HttpMethod.redirectsToGet(method)) {//如果请求方法不是PROPFIND重定向为get请求并去除请求体
            requestBuilder.method("GET", null);
          } else {//请求方法是PROPFIND，加入请求体并保持原有请求方法
            RequestBody requestBody = maintainBody ? userResponse.request().body() : null;
            requestBuilder.method(method, requestBody);
          }
          if (!maintainBody) {//如果没有请求体则移除对应头部字段
            requestBuilder.removeHeader("Transfer-Encoding");
            requestBuilder.removeHeader("Content-Length");
            requestBuilder.removeHeader("Content-Type");
          }
        }

        if (!sameConnection(userResponse, url)) {//不是同一个连接移除Authorization字段
          requestBuilder.removeHeader("Authorization");
        }

        return requestBuilder.url(url).build();
```

可以看到在返回码307或308的时候，只有请求方法不是get不是head才会重定向，300、301、302、303都会重定向，先判断客户端是否允许重定向client#followRedirects()，默认是允许的，然后拿到response中头部location的值作为重定向地址，然后对url做了一些判断，最关键的来了**如果你的请求不是get不是head，我们就假设是post则他会重定向为get请求并且把请求体丢了，对你没有看错基本等于是重定向一律转为get请求并且把请求体丢了。**其实http协议中规定的是301、302禁止将post转为get的，但是由于大部分浏览器都没有遵守都是转为了get，所以okhttp也沿用了这个做法。

## 3. 总结

简单总结下RetryAndFollowUpInterceptor做了3件大事

1. 创建StreamAllocation（这个后面细说）
2. 请求发生错误，判断能不能恢复，如果可以进行重试
3. 根据响应码进行区分处理，我们最常用的重定向就是在这里处理，需要注意的是重定向一律转为get请求并丢弃了请求体


