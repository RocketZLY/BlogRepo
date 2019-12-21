# OkHttp-CacheInterceptor源码解析

> 本文基于okhttp3.10.0

## 1. 主要功能

主要功能其实就一句话按照http协议实现响应的缓存，那既然是按http协议去实现，我们先简单过一下http缓存这块。

## 1.1 Http缓存

http缓存主要靠一些头部去标识

### 1.1.1 Expires

Expires的值为到期时间，即下次请求的时候如果请求时间小于到期时间则可以直接使用缓存

`Expires: Tue, 28 Sep 2004 23:59:59 GMT`

不过Expires 是 HTTP 1.0 的产物，在 HTTP 1.1 中用 Cache-Control 替代，okhttp也是优先取CacheControl的max-age值没有才会取Expires

### 1.1.2 Cache-Control

Cache-Control 的取值有 private、public、no-cache、max-age，no-store 等，默认为private

![](http://rocketzly.androider.top/CacheControl%E5%B1%9E%E6%80%A7.png)

举个板栗

`Cache-Control: private, max-age=0, no-cache`

### 1.1.3 Last-Modified & If-Modified-Since

Last-Modified：响应头中的字段，用来告诉客户端资源最后修改时间

If-Modified-Since：再次请求服务器时，通过此字段通知服务器客户端缓存Respose的Last-Modified值，服务器收到请求后发现有If-Modified-Since 则与被请求资源的最后修改时间进行比对

- 若资源的最后修改时间大于 If-Modified-Since ，说明资源又被改动过，则响应整片资源内容，返回状态码 200
- 若资源的最后修改时间小于或等于 If-Modified-Since ，说明资源无新修改，则响应 HTTP 304，告知浏览器继续使用所保存的 Cache

### 1.1.4 Etag & If-None-Match

Etag：服务器响应请求时，告诉浏览器当前资源在服务器的唯一标识（生成规则由服务器决定）

If-None-Match：再次请求服务器时，通过此字段通知服务器客户段缓存数据的唯一标识，服务器收到请求后发现有头 If-None-Match 则与被请求资源的唯一标识进行比对

- 不同，说明资源又被改动过，则响应整片资源内容，返回状态码 200
- 相同，说明资源无新修改，则响应 HTTP 304，告知浏览器继续使用所保存的 Cache

### 1.1.5 Http缓存流程

![](http://rocketzly.androider.top/http%E7%BC%93%E5%AD%98%E6%B5%81%E7%A8%8B.png)

- 有没有缓存
  - 没有，则进行网络请求并根据response缓存控制字段看是否缓存本次响应
  - 有
    - 判断是否过期
      - 未过期直接使用
      - 过期
        - 缓存有Etag么，有的话将etag值通过If-None-Matcher传递给服务端看缓存是否有修改，没有则返回304继续使用缓存并更新缓存过期时间，否则返回200和新的资源数据并根据response缓存控制字段看是否缓存本次响应
        - 缓存有Last-Modified么，有则通过If-Modified-Since把缓存的修改时间告知服务器，如果在这个时间后资源没有更新，则返回304继续使用缓存并更新缓存过期时间，否则返回200和新的资源数据并根据response缓存控制字段看是否缓存本次响应
        - 既没有Etag也没有Last-Modified则直接进行请求并根据response缓存控制字段看是否缓存本次响应

## 2. 源码解析

由于源码有点多，我们先粗读一遍拦截器的源码知道是干嘛的，然后在细说下CacheStrategy中对于http缓存的处理

## 2.1 CacheInterceptor#intercept()

```java
  @Override public Response intercept(Chain chain) throws IOException {
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;//获取本地缓存的Resp赋值为cacheCandidate字段

    long now = System.currentTimeMillis();

    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();//创建缓存策略工厂，其内部就是判断是否可以使用缓存，缓存是否过期等一系列操作最终决定是使用缓存还是进行网络请求
    Request networkRequest = strategy.networkRequest;// 若是不为 null ，表示需要进行网络请求
    Response cacheResponse = strategy.cacheResponse;// 若是不为 null ，表示可以使用本地缓存

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {//本地有缓存，但不可用关闭掉
      closeQuietly(cacheCandidate.body()); 
    }

    if (networkRequest == null && cacheResponse == null) {//既没有缓存也不进行网络请求返回resp code为504
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    if (networkRequest == null) {//如果网络请求为null而缓存不为null则直接使用缓存
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
		//到这里还剩
    //networkRequest!=null&&cacheResponse==null
    //networkRequest!=null&&cacheResponse!=null
    //两种情况
    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);//进行网络请求
    } finally {
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

   
    if (cacheResponse != null) {//如果缓存不为null再看resp返回码
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {//判断响应码为304代表资源未更新，则刷新缓存头部、时间等信息
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);//更新缓存
        return response;
      } else {//否则关闭缓存
        closeQuietly(cacheResponse.body());
      }
    }
		//到这里就还有
    //networkRequest!=null&&cacheResponse==null
    //networkRequest!=null&&cacheResponse!=null&&返回码不为304
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {//cache不为null
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {//判断当前响应有body并且需要缓存
        CacheRequest cacheRequest = cache.put(response);//进行缓存
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {//不能进行缓存
        try {
          cache.remove(networkRequest);//移除缓存
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```

大体流程是先通过request从本地取对应的response缓存，然后通过CacheStrategy#get()确定是进行网络请求还是使用缓存，如果既不使用缓存也不进行网络请求返回一个code为504的response，如果网络请求不可用缓存可用直接使用缓存，如果网络请求可用进行请求，请求后在判断有缓存并且响应码为304的话则更新缓存，否则在判断reponse能否缓存，可以的话则存储缓存，不可以的话则删除缓存。

## 2.2 CacheStrategy

接下来看到缓存策略的代码

```java
CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get()
```

### 2.2.1 new CacheStrategy.Factory()

第一步先看new CacheStrategy.Factory(now, chain.request(), cacheCandidate)

```java
    public Factory(long nowMillis, Request request, Response cacheResponse) {
      this.nowMillis = nowMillis;//存储了当前时间
      this.request = request;//request
      this.cacheResponse = cacheResponse;//本地获取的resp

      if (cacheResponse != null) {//本地获取的resp不为null
        this.sentRequestMillis = cacheResponse.sentRequestAtMillis();//读取请求发送时间
        this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();//缓存resp接收时间
        Headers headers = cacheResponse.headers();
        for (int i = 0, size = headers.size(); i < size; i++) {
          String fieldName = headers.name(i);
          String value = headers.value(i);
          if ("Date".equalsIgnoreCase(fieldName)) {//读取响应体的Date
            servedDate = HttpDate.parse(value);
            servedDateString = value;
          } else if ("Expires".equalsIgnoreCase(fieldName)) {//读取响应体的Expires
            expires = HttpDate.parse(value);
          } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {//读取响应体的Last-Modified
            lastModified = HttpDate.parse(value);
            lastModifiedString = value;
          } else if ("ETag".equalsIgnoreCase(fieldName)) {//读取响应体的ETag
            etag = value;
          } else if ("Age".equalsIgnoreCase(fieldName)) {//读取响应体的Age
            ageSeconds = HttpHeaders.parseSeconds(value, -1);
          }
        }
      }
    
```

在构造方法的时候读取了些必要信息缓存起来，对于header中字段我们细说下

- Date：创建报文的日期时间

- Expires：实体主体过期的日期时间

- Last-Modified：资源的最后修改日期时间

- ETag：资源的匹配信息、生成规则由服务端确定

- Age：消息头里包含消息对象在缓存代理中存贮的时长，以秒为单位

  Age消息头的值通常接近于0。表示此消息对象刚刚从原始服务器获取不久；其他的值则是表示代理服务器当前的系统时间与此应答消息中的通用消息头 Date 的值之差。

### 2.2.2 CacheStrategy.Facory#get()

```java
    public CacheStrategy get() {
      CacheStrategy candidate = getCandidate();//根据一定策略创建CacheStrategy

      if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {//如果要进行网络请求而请求头部CacheControl中又有only-if-cached标识则丢弃本次
        return new CacheStrategy(null, null);
      }

      return candidate;
    }
```

可以看到关键是在getCandidate()去创建CacheStrategy，那么我们先看下构造方法

```java
  public final @Nullable Request networkRequest;
  public final @Nullable Response cacheResponse;

  CacheStrategy(Request networkRequest, Response cacheResponse) {
    this.networkRequest = networkRequest;
    this.cacheResponse = cacheResponse;
  }
```

构造中将cacheResponse和networkRequest用成员变量存储，后面再CacheInterceptor#intercept()中通过networkRequest和cacheResponse是否为null来判断是进行网络请求还是走缓存。然后在看下getCandidate()如何获取的CacheStrategy实例

```java
    private CacheStrategy getCandidate() {
      if (cacheResponse == null) {//如果没有缓存
        return new CacheStrategy(request, null);//直接创建有请求没缓存的CacheStrategy
      }

      if (request.isHttps() && cacheResponse.handshake() == null) {//如果请求是https而缓存Response握手失败直接进行网络请求
        return new CacheStrategy(request, null);
      }

      if (!isCacheable(cacheResponse, request)) {//如果不能进行缓存直接进行网络请求，具体逻辑下面细说
        return new CacheStrategy(request, null);
      }

      CacheControl requestCaching = request.cacheControl();
      if (requestCaching.noCache() || hasConditions(request)) {//如果request的CacheControl中有noCache或者头部有(If-Modified-Since||If-None-Match)进行网络请求
        return new CacheStrategy(request, null);
      }

      CacheControl responseCaching = cacheResponse.cacheControl();
      if (responseCaching.immutable()) {//如果resp中的CacheControl中有immutable则直接使用缓存
        return new CacheStrategy(null, cacheResponse);
      }

      long ageMillis = cacheResponseAge();//获取当前缓存使用时长
      long freshMillis = computeFreshnessLifetime();//获取缓存可用时间

      if (requestCaching.maxAgeSeconds() != -1) {
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));//取request和response中maxAge最小值作为缓存可用时间
      }

      long minFreshMillis = 0;//min-fresh指示客户端希望响应至少在指定的秒数内仍保持新鲜。
      if (requestCaching.minFreshSeconds() != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());//获取min-fresh值
      }

      long maxStaleMillis = 0;//max-stale表示客户端愿意接受超过其到期时间的响应。
      if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {//没有must-revalidate并且max-stale有值即愿意接受过期缓存但过期时间未超过max-stale
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
      }

      if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {//如果CacheControl没有noCache并且当前缓存可用
        Response.Builder builder = cacheResponse.newBuilder();
        if (ageMillis + minFreshMillis >= freshMillis) {//缓存已经过期但是可用添加警告头部
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
        return new CacheStrategy(null, builder.build());//返回使用缓存的CacheStrategy实例
      }
			//到这里则是有缓存但不能使用
      String conditionName;
      String conditionValue;
      if (etag != null) {//优先用etag进行缓存有效性的判断
        conditionName = "If-None-Match";
        conditionValue = etag;
      } else if (lastModified != null) {//其次才使用lastModified进行缓存有效性的判断
        conditionName = "If-Modified-Since";
        conditionValue = lastModifiedString;
      } else if (servedDate != null) {//如果都没有默认使用Date作为lastModified
        conditionName = "If-Modified-Since";
        conditionValue = servedDateString;
      } else {//都没有的话直接进行网络请求
        return new CacheStrategy(request, null); // No condition! Make a regular request.
      }

      Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
      Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);//将对应字段添加到头部

      Request conditionalRequest = request.newBuilder()
          .headers(conditionalRequestHeaders.build())
          .build();
      return new CacheStrategy(conditionalRequest, cacheResponse);//返回一个有缓存但需要进行网络请求的CacheStrategy
    }
```

总结下上面代码，策略一共可以分为三类

1. 缓存不可用进行网络请求
   1. 缓存为null
   2. 请求为https，但是缓存的response没有握手
   3. 缓存不可用（这个下面会细说）
   4. 请求头的CacheControl中有noCache，或者请求中有If-Modified-Since和If-None-Match任意一个
2. 使用缓存
   1. response的CacheControl中有immutable
   2. response的CacheControl中没有noCache并且缓存未过期
3. 缓存过期使用网络请求，并在请求头中带上过期缓存资源标识优先使用etag、没有的话才使用lastModified。如果资源未更新的话，服务端会返回304告诉客户端继续使用缓存并刷新缓存过期时间等信息、否则返回200和完整的资源数据覆盖原有缓存。

接下来看刚刚遗留的两个点缓存可用的判断和缓存过期时间的计算

### 2.2.3 CacheStrategy#isCacheable()

```java
  public static boolean isCacheable(Response response, Request request) {
    switch (response.code()) {
      case HTTP_OK:
      case HTTP_NOT_AUTHORITATIVE:
      case HTTP_NO_CONTENT:
      case HTTP_MULT_CHOICE:
      case HTTP_MOVED_PERM:
      case HTTP_NOT_FOUND:
      case HTTP_BAD_METHOD:
      case HTTP_GONE:
      case HTTP_REQ_TOO_LONG:
      case HTTP_NOT_IMPLEMENTED:
      case StatusLine.HTTP_PERM_REDIRECT:
        //这些响应码可以被缓存只要响应头中没说不能缓存
        break;

      case HTTP_MOVED_TEMP:
      case StatusLine.HTTP_TEMP_REDIRECT://302和307需要判断下CacheControl有没有需要缓存属性
        if (response.header("Expires") != null
            || response.cacheControl().maxAgeSeconds() != -1
            || response.cacheControl().isPublic()
            || response.cacheControl().isPrivate()) {
          break;
        }
        // Fall-through.

      default://其他情况不能被缓存
        return false;
    }

    return !response.cacheControl().noStore() && !request.cacheControl().noStore();//请求和响应头的CacheControl中没有noStore则可以缓存
  }
```

从代码可以看出大部分情况下都是通过请求和响应头的CacheControl中没有noStore值进行判断能否缓存

### 2.2.4 缓存过期时间的计算

这里我们剔除无用代码只看缓存时间相关的代码

```java
private CacheStrategy getCandidate() {  
			long ageMillis = cacheResponseAge();//计算缓存使用时长
      long freshMillis = computeFreshnessLifetime();//缓存过期时间
      if (requestCaching.maxAgeSeconds() != -1) {//获取request和response中CacheControl的max-age中最小的作为过期时间
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
      }
      long minFreshMillis = 0;//min-fresh客户端希望响应至少在指定的秒数内仍保持新鲜
      if (requestCaching.minFreshSeconds() != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
      }
      long maxStaleMillis = 0;//max-stale表示客户端愿意接受过期的响应但是过期时间未超过max-stale的响应
      if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {//must-revalidate指示一旦资源变得陈旧，缓存就不能使用。所以如果没有must-revalidate并且max-stale不为-1则获取对应值
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
      }
 if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {//响应的CacheControl中没有noCache 并且 缓存使用时长+客户端希望缓存保持新鲜的最小时间<缓存可用时间+过期后还能使用最大时间
        //使用缓存
      }
  }
```

**也就是通过缓存使用时长 + 客户端希望服务端保持缓存新鲜最小时长 < 缓存可用时间 + 过期后缓存还能使用最大时间判断缓存是否可用。**

```java
    private long cacheResponseAge() {
      long apparentReceivedAge = servedDate != null
          ? Math.max(0, receivedResponseMillis - servedDate.getTime())
          : 0;
      long receivedAge = ageSeconds != -1//拿到缓存在代理保存的时间
          ? Math.max(apparentReceivedAge, SECONDS.toMillis(ageSeconds))
          : apparentReceivedAge;
      long responseDuration = receivedResponseMillis - sentRequestMillis;//发送到接受响应的耗时
      long residentDuration = nowMillis - receivedResponseMillis;//缓存在本地的保存时间
      return receivedAge + responseDuration + residentDuration;
    }
```

**而缓存使用时长 = 客户端发送请求到接收响应的耗时 + 代理服务器缓存时间 + 响应在客户端本地保存时间**

```java
    private long computeFreshnessLifetime() {
      CacheControl responseCaching = cacheResponse.cacheControl();
      if (responseCaching.maxAgeSeconds() != -1) {//读取CacheControl的max-age
        return SECONDS.toMillis(responseCaching.maxAgeSeconds());
      } else if (expires != null) {//其次读取expires
        long servedMillis = servedDate != null
            ? servedDate.getTime()
            : receivedResponseMillis;
        long delta = expires.getTime() - servedMillis;
        return delta > 0 ? delta : 0;
      } else if (lastModified != null
          && cacheResponse.request().url().query() == null) {
        long servedMillis = servedDate != null
            ? servedDate.getTime()
            : sentRequestMillis;
        long delta = servedMillis - lastModified.getTime();
        return delta > 0 ? (delta / 10) : 0;
      }
      return 0;
    }
```

**缓存可用时间的优先读取max-age，如果没有在看expires，如果也没有但lastModified不为null并且url不包含query则默认取报文创建时间减去资源修改时间的差值除以10作为缓存可用时间**

## 3. 总结

CacheInterceptor主要是用于http缓存的操作，先从本地拿出缓存，然后在通过CacheStrategy.Facory#get()获取具体的缓存策略，如果既不使用缓存也不进行网络请求返回一个code为504的response，如果网络请求不可用缓存可用直接使用缓存，如果需要网络请求进行请求，请求后在判断有缓存并且响应码为304的话则更新缓存，否则在判断reponse能否缓存，可以的话则存储缓存，不可以的话则删除缓存，具体细节上面已经分析过这里不在细说，唯一还有点没说的是如何缓存的，后面要是有时间再来篇博客介绍。




