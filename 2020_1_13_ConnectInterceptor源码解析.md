# ConnectInterceptor源码解析

> 本文基于okhttp3.10.0

## 1. 概述

ConnectInterceptor主要是用于建立连接，并再连接成功后将流封装成对象传递给下一个拦截器CallServerInterceptor与远端进行读写操作。

这个过程中会涉及比较多类我们简述下每个类的作用

- StreamAllocation：类似一个工厂用来创建连接RealConnection和与远端通信的流的封装对象HttpCodec
- ConnectionPool：连接池用来存储可用的连接，在条件符合的情况下进行连接复用
- HttpCodec：对输入输出流的封装对象，对于http1和2实现不同
- RealConnection：对tcp连接的封装

## 2. 源码解析

```java
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;

  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```

可以看到代码非常少，主要就是从realChain中拿出StreamAllocation然后调用StreamAllocation#newStream()获取流的封装对象HttpCodec、StreamAllocation#connection()拿到连接对象RealConnection，然后将它们传递给下一个拦截器。

很显然主要逻辑是在StreamAllocation中实现的，而StreamAllocation是在第一个拦截器retryAndFollowUpInterceptor创建的直到ConnectInterceptor才使用

```java
#RetryAndFollowUpInterceptor
@Override public Response intercept(Chain chain) throws IOException {
		    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    ...
}
```

可以看到构造方法一共传入了5个参数，我们只需要关注前三个即可

- client.connectionPool()：okhttp连接池
- createAddress(request.url())：将url解析为Address对象
- call：请求对象Call

### 2.1 ConnectionPool

连接池通过OkHttpClient实例的connectionPool()方法获取，默认初始化是在OkHttpClient.Builder构造函数

**主要功能是用来缓存连接，当符合条件的时候进行连接复用，内部通过一个队列去缓存连接，当超过缓存时间后会自动清理过期连接。**

先来看下ConnectionPool构造方法

```java
private final int maxIdleConnections;//最大闲置连接数
private final long keepAliveDurationNs;//每个连接最大缓存时间

public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);//默认最大缓存5个闲置连接，过期时间5分钟
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    // Put a floor on the keep alive duration, otherwise cleanup will spin loop.
    if (keepAliveDuration <= 0) {
      throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
  }
```

默认最大缓存5个连接，过期时间为5分钟，存储连接是通过ConnectionPool#put()方法

```java
private final Deque<RealConnection> connections = new ArrayDeque<>();//缓存队列
boolean cleanupRunning;//开启清理过期连接任务标记位
private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));

void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);//在线程池中开启清理过期连接任务
    }
    connections.add(connection);//添加到缓存队列
  }
```

存储连接的时候，如果没开启清理任务则开启清理过期连接任务并缓存新的连接

```java
  private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
        long waitNanos = cleanup(System.nanoTime());//获取最近一个即将过期连接的倒计时
        if (waitNanos == -1) return;//-1则代表没有缓存连接了直接return
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);//wait最近一个即将过期连接的倒计时后在进行检测
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };
```

清理过期连接runnable则是通过cleanup()方法获取最近一个即将过期连接的倒计时，为-1则代表缓存已经清空了直接return退出，否则wait一个即将超时的时间后在进行检查

```java
  long cleanup(long now) {
    int inUseConnectionCount = 0;//使用的连接数
    int idleConnectionCount = 0;//闲置的连接数
    RealConnection longestIdleConnection = null;//闲置时间最长的连接
    long longestIdleDurationNs = Long.MIN_VALUE;

    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();//遍历缓存的连接

        if (pruneAndGetAllocationCount(connection, now) > 0) {//如果该连接有人使用
          inUseConnectionCount++;//使用连接数++
          continue;
        }

        idleConnectionCount++;//否则闲置连接数++

        long idleDurationNs = now - connection.idleAtNanos;//当前连接闲置时间
        if (idleDurationNs > longestIdleDurationNs) {//找到闲置最久的连接
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {//闲置最久的连接时间超过5分钟或者闲置连接数大于5
        connections.remove(longestIdleConnection);//移除当前连接
      } else if (idleConnectionCount > 0) {//闲置连接数大于0
        return keepAliveDurationNs - longestIdleDurationNs;//return缓存最久连接剩余过期时间
      } else if (inUseConnectionCount > 0) {//如果使用连接数大于0
        return keepAliveDurationNs;//return最大缓存时间
      } else {//没有缓存连接了
        cleanupRunning = false;//把标记位置为false
        return -1;//返回-1停止检测
      }
    }
```

遍历缓存的连接，找到闲置最久的连接，和使用与闲置连接数量

1. 当闲置最久的连接闲置时间大于最大闲置时间或者闲置连接数大于最大闲置连接数，直接移除该连接
2. 当闲置连接数大于0则计算闲置最久的连接过期时间作为倒计时返回
3. 当使用连接数大于0，返回最大缓存时间作为倒计时
4. 没有缓存连接了把标志位cleanupRunning置为false并返回-1停止清理任务。

这里需要注意下的是对于连接是否在使用的判断pruneAndGetAllocationCount()方法

```java
  private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<StreamAllocation>> references = connection.allocations;//拿到RealConnection中StreamAllocation引用
    for (int i = 0; i < references.size(); ) {//遍历
      Reference<StreamAllocation> reference = references.get(i);

      if (reference.get() != null) {//如果持有StreamAllocation则代表有使用
        i++;
        continue;
      }

      references.remove(i);//如果reference.get() == null则代表内存泄露了，移除该引用
      connection.noNewStreams = true;//该连接不再创建新的流

      if (references.isEmpty()) {//如果StreamAllocation引用为空
        connection.idleAtNanos = now - keepAliveDurationNs;//将当前闲置时间设置为now - keepAliveDurationNs即满足清除条件会被直接从缓存移除
        return 0;
      }
    }

    return references.size();//返回当前连接使用数
  }
```

对于连接是否在使用是通过RealConnection中的成员变量allocations来判断，如果有请求正在使用则会将StreamAllocation对象的引用存储到allocations中，具体存储的代码是在StreamAllocation中后面会说到

看完put再来看get

```java
  @Nullable RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {//遍历缓存连接
      if (connection.isEligible(address, route)) {//满足条件
        streamAllocation.acquire(connection, true);//复用连接
        return connection;
      }
    }
    return null;
  }
```

遍历缓存连接，如果满足条件则复用该连接

```java
  public boolean isEligible(Address address, @Nullable Route route) {
    if (allocations.size() >= allocationLimit || noNewStreams) return false;//如果当前连接流使用数>=最大并发或者不能创建新流了则return false

    if (!Internal.instance.equalsNonHost(this.route.address(), address)) return false;//地址中除了host有不相同的返回false

    if (address.url().host().equals(this.route().address().url().host())) {//如果host相同则返回true
      return true; 
    }

    if (http2Connection == null) return false;//如果上面条件都不满足并且不是http2则返回false http2不再本次源码讨论范围下面就不分析了
    if (route == null) return false;
    if (route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (this.route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (!this.route.socketAddress().equals(route.socketAddress())) return false;

    if (route.address().hostnameVerifier() != OkHostnameVerifier.INSTANCE) return false;
    if (!supportsUrl(address.url())) return false;

    try {
      address.certificatePinner().check(address.url().host(), handshake().peerCertificates());
    } catch (SSLPeerUnverifiedException e) {
      return false;
    }
```

可以看到在满足如下三个条件的时候http1连接可以服用

1. 小于当前连接最大并发数，http1最大并发为1并且可以创建新的流
2. 请求路径中除了host以外的全部相同
3. host也相同

满足以上三个条件即可复用缓存的连接

复用是通过streamAllocation.acquire(connection, true)方法实现

```java
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```

即通过成员变量存储connection，并connection.allocations.add(new StreamAllocationReference(this, callStackTrace));给当前连接添加上StreamAllocation引用代表有一个流正在使用该连接。

那么总结下ConnectionPool是用来缓存http连接的，当条件满足的时候复用连接，默认每个连接最大缓存时长5分钟，最大缓存数量5个。

### 2.2 Address

在RetryAndFollowUpInterceptor#createAddress()方法创建

```java
  private Address createAddress(HttpUrl url) {
    SSLSocketFactory sslSocketFactory = null;
    HostnameVerifier hostnameVerifier = null;
    CertificatePinner certificatePinner = null;
    if (url.isHttps()) {//请求是https
      sslSocketFactory = client.sslSocketFactory();
      hostnameVerifier = client.hostnameVerifier();
      certificatePinner = client.certificatePinner();
    }

    return new Address(url.host(), url.port(), client.dns(), client.socketFactory(),
        sslSocketFactory, hostnameVerifier, certificatePinner, client.proxyAuthenticator(),
        client.proxy(), client.protocols(), client.connectionSpecs(), client.proxySelector());
  }
```

当是https的时候SSLSocketFactory、HostnameVerifier、CertificatePinner才会被赋值，后面会通过SSLSocketFactory ！= null这个判断请求是不是https，构造的时候只有host和port是从url中取得，其他都是在OkHttpClient的配置。

```java
  public Address(String uriHost, int uriPort, Dns dns, SocketFactory socketFactory,
      @Nullable SSLSocketFactory sslSocketFactory, @Nullable HostnameVerifier hostnameVerifier,
      @Nullable CertificatePinner certificatePinner, Authenticator proxyAuthenticator,
      @Nullable Proxy proxy, List<Protocol> protocols, List<ConnectionSpec> connectionSpecs,
      ProxySelector proxySelector) {
    this.url = new HttpUrl.Builder()
        .scheme(sslSocketFactory != null ? "https" : "http")
        .host(uriHost)
        .port(uriPort)
        .build();

    if (dns == null) throw new NullPointerException("dns == null");
    this.dns = dns;

    if (socketFactory == null) throw new NullPointerException("socketFactory == null");
    this.socketFactory = socketFactory;

    if (proxyAuthenticator == null) {
      throw new NullPointerException("proxyAuthenticator == null");
    }
    this.proxyAuthenticator = proxyAuthenticator;

    if (protocols == null) throw new NullPointerException("protocols == null");
    this.protocols = Util.immutableList(protocols);

    if (connectionSpecs == null) throw new NullPointerException("connectionSpecs == null");
    this.connectionSpecs = Util.immutableList(connectionSpecs);

    if (proxySelector == null) throw new NullPointerException("proxySelector == null");
    this.proxySelector = proxySelector;

    this.proxy = proxy;
    this.sslSocketFactory = sslSocketFactory;
    this.hostnameVerifier = hostnameVerifier;
    this.certificatePinner = certificatePinner;
  }
```

可以看到构造函数只是将传入的属性通过成员变量存储起来。

总结下Address就是将url分解，然后通过成员变量存储不同的数据供后面使用。

### 2.3 StreamAllocation

上面把两个重要的对象说完了，那么回到ConnectInterceptor#intercept

```java
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    boolean doExtensiveHealthChecks = !request.method().equals("GET");//非get请求需要额外的检查
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);//获取流的封装对象
    RealConnection connection = streamAllocation.connection();//获取连接

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

调用StreamAllocation#newStream()获取流的封装对象HttpCodec、调用StreamAllocation#connection()拿到连接对象RealConnection，然后将它们传递给下一个拦截器。

```java
  public synchronized RealConnection connection() {
    return connection;//获取连接
  }  

	public HttpCodec newStream(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);//找到一个可用的连接
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);//创建流封装对象

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```

StreamAllocation#connection()获取成员变量connection返回，主要流程是在newStream()获取可用的连接并创建一个流的封装对象

```java
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);//找一个可用的连接

        synchronized (connectionPool) {
        if (candidate.successCount == 0) {//如果successCount == 0代表是个新连接直接返回
          return candidate;
        }
      }

      if (!candidate.isHealthy(doExtensiveHealthChecks)) {//不为新连接则检查连接是否可用
        noNewStreams();//不可用释放连接
        continue;
      }

      return candidate;
    }
  }
```

while循环找到一个可用的连接则返回，如果是新连接则直接返回，否则检查连接是否可用，不可用的释放连接。

```java
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {//1.连接不为空则直接使用
        result = this.connection;
        releasedConnection = null;
      }

      if (result == null) {//2.从连接池中找是否有可以复用的连接
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);
    if (result != null) {
      return result;
    }

    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {//切换新的路由
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {//3.用新的路由再次看能不能复用连接
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);//4.创建新的连接
        acquire(result, false);//成员变量记录下来
      }
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);//建立tcp连接
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // Pool the connection.
      Internal.instance.put(connectionPool, result);//缓存连接
    }
    closeQuietly(socket);
    return result;
  }
```

这段我简化了不少代码，主要流程分为四步

1. 如果当前StreamAllocation有Connection成员变量则直接使用（这个对应的是重定向的url路径和当前请求匹配的情况下）
2. 从连接池中找可以复用的连接
3. 切换路由，再次从连接池中找复用连接（关于路由这块说实话我不太懂，哪位大佬懂的话欢迎指导）
4. 创建新的连接

对于第一个情况我们简单说下StreamAllocation是在RetryAndFollowUpInterceptor拦截器创建的，也就是说每次请求都会创建，所以大多数情况下StreamAllocation#connection成员变量为null，但是当收到response为重定向的时候RetryAndFollowUpInterceptor的while循环中会调用sameConnection()判断重定向的请求和当前请求连接是否相同如果相同则不会再创建StreamAllocation，也就会进入我们上面第一个条件了。

```java
  private boolean sameConnection(Response response, HttpUrl followUp) {
    HttpUrl url = response.request().url();
    return url.host().equals(followUp.host())
        && url.port() == followUp.port()
        && url.scheme().equals(followUp.scheme());
  }
```

即满足host相同port相同scheme相同则可以使用同一个连接，条件还是比较苛刻的。

回到正题如果连接是新创建的则

1. StreamAllocation#acquire()，streamAllocation记录下连接，给connection.allocations添加streamAllocation引用
2. RealConnection#connect()建立tcp连接
3. 缓存到连接池

```java
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;//记录connection
    this.reportedAcquired = reportedAcquired;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));//给connection.allocations添加streamAllocation引用代表有请求在使用该连接
  }
```

connection.allocations.add(new StreamAllocationReference(this, callStackTrace));给当前连接添加上StreamAllocation引用代表有一个流正在使用该连接。

### 2.4 RealConnection

```java
  public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled, Call call,
      EventListener eventListener) {
    while (true) {
      try {
        if (route.requiresTunnel()) {//如果是隧道请求
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);//建立隧道
          if (rawSocket == null) {
            break;
          }
        } else {
          connectSocket(connectTimeout, readTimeout, call, eventListener);//建立http连接
        }
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);//完成协议操作比如https
        eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
        break;
      } catch (IOException e) {
        ....
      }
    }

    if (route.requiresTunnel() && rawSocket == null) {
      ProtocolException exception = new ProtocolException("Too many tunnel connections attempted: "
          + MAX_TUNNEL_ATTEMPTS);
      throw new RouteException(exception);
    }

    if (http2Connection != null) {//如果是http2请求
      synchronized (connectionPool) {
        allocationLimit = http2Connection.maxConcurrentStreams();//修改当前连接支持的最大并发数
      }
    }
  }
```

如果请求需要建立隧道则建立隧道，否则就建立普通的tcp连接，然后进行协议相关操作，如果是http2的话会修改当前连接支持的最大并发数，默认为1。

```java
  private void connectSocket(int connectTimeout, int readTimeout, Call call,
      EventListener eventListener) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();

    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);//创建Socket对象

    rawSocket.setSoTimeout(readTimeout);//设置读取数据时阻塞链路的超时时间。
    try {
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);//调用 socket.connect()建立连接
    } catch (ConnectException e) {
      ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
      ce.initCause(e);
      throw ce;
    }
    try {
      //拿到流对象
      source = Okio.buffer(Okio.source(rawSocket));
      sink = Okio.buffer(Okio.sink(rawSocket));
    } catch (NullPointerException npe) {
      if (NPE_THROW_WITH_NULL.equals(npe.getMessage())) {
        throw new IOException(npe);
      }
    }
  }
```

可以看到内部封装的是socket对象，然后通过socket.connect()进行连接，通过成员变量将输入输出流存储起来

```java
  private void establishProtocol(ConnectionSpecSelector connectionSpecSelector,
      int pingIntervalMillis, Call call, EventListener eventListener) throws IOException {
    if (route.address().sslSocketFactory() == null) {//如果不是https请求的话
      protocol = Protocol.HTTP_1_1;
      socket = rawSocket;
      return;
    }

    eventListener.secureConnectStart(call);
    connectTls(connectionSpecSelector);//建立ssl通道
    eventListener.secureConnectEnd(call, handshake);

    if (protocol == Protocol.HTTP_2) {//如果是http2请求
      socket.setSoTimeout(0); //创建http2实例
      http2Connection = new Http2Connection.Builder(true)
          .socket(socket, route.address().url().host(), source, sink)
          .listener(this)
          .pingIntervalMillis(pingIntervalMillis)
          .build();
      http2Connection.start();
    }
  }

```

这里就看到前面说的通过route.address().sslSocketFactory() == null判断当前请求是不是https请求，不是的话建立ssl通道，如果是http2的话创建Http2Connection实例，后面也会通过这个实例是否为空判断当前请求是不是http2

现在知道RealConnection怎么创建的了再来看RealConnection#newCodec()

```java
  public HttpCodec newCodec(OkHttpClient client, Interceptor.Chain chain,
      StreamAllocation streamAllocation) throws SocketException {
    if (http2Connection != null) {//如果是http2则创建ttp2Codec
      return new Http2Codec(client, chain, streamAllocation, http2Connection);
    } else {//否则创建Http1Codec
      socket.setSoTimeout(chain.readTimeoutMillis());
      source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
      return new Http1Codec(client, streamAllocation, source, sink);
    }
  }
```

可以发现就是判断http2Connection实例是否不为null来判断当前请求是不是http2，一般情况下都是http1，也就是返回一个Http1Codec实例而Http1Codec和Http2Codec都是HttpCodec的实现类

```java
public interface HttpCodec {
  int DISCARD_STREAM_TIMEOUT_MILLIS = 100;
  Sink createRequestBody(Request request, long contentLength);
  void writeRequestHeaders(Request request) throws IOException;
  void flushRequest() throws IOException;
  void finishRequest() throws IOException;
  Response.Builder readResponseHeaders(boolean expectContinue) throws IOException;
  ResponseBody openResponseBody(Response response) throws IOException;
  void cancel();
}
```

可以看到都是对流的操作，你猜的没错HttpCodec就是对流的再一次封装。

到这里ConnectInterceptor基本就分析完了

```java
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

获取可用的连接RealConnection和对流的封装HttpCodec传递给下个拦截器，完成与远端流的读写相关操作。

## 3. 总结

最后再来总结下ConnectInterceptor几个对象干嘛的

- StreamAllocation：类似一个工厂用来创建连接RealConnection和与远端通信的流的封装对象HttpCodec
- Address：将url分解，然后通过成员变量存储不同的数据供其他地方使用
- ConnectionPool：连接池用来存储可用的连接，在条件符合的情况下进行连接复用
- HttpCodec：对输入输出流的封装对象，对于http1和2实现不同，方便后面拦截器使用
- RealConnection：对tcp连接的封装对象

对于ConnectInterceptor我觉得最特殊的地方就是有连接池缓存连接做连接复用












