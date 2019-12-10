# BridgeInterceptor源码解析

> 本文基于okhttp3.10.0

这是okhttp拦截器中第二个，作用非常简单就是增加必要的请求头，处理请求体。

## 1. 主要功能

1. 添加请求头
2. Cookie管理
3. Gzip压缩

## 1.1 添加请求体

```java
  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    if (body != null) {//body不为空
      MediaType contentType = body.contentType();
      if (contentType != null) {//contentType不为空
        requestBuilder.header("Content-Type", contentType.toString());//添加contentType参数
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {//body长度不为空
        requestBuilder.header("Content-Length", Long.toString(contentLength));//添加Content-Length参数
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");//分块传输
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {//host为null增加host参数
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {//默认支持keep-alive
      requestBuilder.header("Connection", "Keep-Alive");
    }

    if (userRequest.header("User-Agent") == null) {//增加用户标识信息
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());//进行请求

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    return responseBuilder.build();//返回结果
  }
```

剔除Cookie和Gzip代码后BridgeInterceptor变的非常简单，这里我们归纳下都加了那些头和作用

- Content-Type：报文主体内容类型
- Content-Length：实体主体的大小（单位：字节）
- Transfer-Encoding：规定了传输报文主体时采用的编码方式
- Host：请求资源所在服务器，段在 HTTP/1.1 规范内是唯一一个必须被包含在请 求内的首部字段
- Connection：管理持久连接
- User-Agent：HTTP 客户端程序的信息

我随便抓了个请求，可以看下它头部信息

![](http://rocketzly.androider.top/%E8%AF%B7%E6%B1%82%E5%A4%B4.png)

## 1.2 Cookie管理

```java
  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());//获取本地存储的cookie
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));//cookies不等于空添加到头部
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());//存储服务端返回的coockie

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    return responseBuilder.build();
  }
```

cookieJar是通过okhttpclient#cookieJar()获取的，默认实现为CookieJar.NO_COOKIES

```java
public interface CookieJar {
  CookieJar NO_COOKIES = new CookieJar() {//默认实现
    @Override public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
    }

    @Override public List<Cookie> loadForRequest(HttpUrl url) {
      return Collections.emptyList();
    }
  };

  void saveFromResponse(HttpUrl url, List<Cookie> cookies);//存储coockie

  List<Cookie> loadForRequest(HttpUrl url);//通过url读取cookie
}
```

所以在读取的时候默认是返回一个空的list，再来看下存储HttpHeaders#receiveHeaders()

```java
  public static void receiveHeaders(CookieJar cookieJar, HttpUrl url, Headers headers) {
    if (cookieJar == CookieJar.NO_COOKIES) return;//默认不存储

    List<Cookie> cookies = Cookie.parseAll(url, headers);//将headers中的cookie解析为Cookie对象
    if (cookies.isEmpty()) return;

    cookieJar.saveFromResponse(url, cookies);//触发存储逻辑
  }
```

默认直接走到了return里，否则通过Cookie#parseAll()解析response的header中的cookie

```java
  public static List<Cookie> parseAll(HttpUrl url, Headers headers) {
    List<String> cookieStrings = headers.values("Set-Cookie");//其实就是取响应头中set-cookie字段的值，判断是cookie值转为cookie对象存储在list中返回
    List<Cookie> cookies = null;

    for (int i = 0, size = cookieStrings.size(); i < size; i++) {
      Cookie cookie = Cookie.parse(url, cookieStrings.get(i));
      if (cookie == null) continue;
      if (cookies == null) cookies = new ArrayList<>();
      cookies.add(cookie);
    }

    return cookies != null
        ? Collections.unmodifiableList(cookies)
        : Collections.<Cookie>emptyList();
  }
```

那么如果要使用cookie只需要实现CookieJar接口并传给okhttpclient即可。

接下来可以看下http请求中关于cookie运用的例子

![](http://rocketzly.androider.top/cookie.png)

![](http://rocketzly.androider.top/cookie%E5%B1%9E%E6%80%A7.png)

okhttp也就是根据这个规则解析respose中的cookie在下次请求的时候在带上cookie信息。

## 1.3 Gzip压缩

```java
  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {//如果没有Accept-Encoding默认使用gzip压缩
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    Response networkResponse = chain.proceed(requestBuilder.build());
    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {//使用的gzip压缩
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));//进行gzip解压还原数据
    }

    return responseBuilder.build();//返回给我们
  }
```

okhttp默认在头部添加Accept-Encoding：gzip，如果服务端确实支持gzip压缩的话，okhttp会帮我们进行解压省去了自行解压的过程。

## 2. 总结

BridgeInterceptor非常简单就不总结了，就酱
