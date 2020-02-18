# OkHttp-CallServerInterceptor源码解析

> 本文基于okhttp3.10.0，并且只介绍http1相关的内容不涉及http2.0

终于到最后一个拦截器了，前面我们说到在ConnectInterceptor中会建立连接创建RealConnection和HttpCodec，然后传递给下一个拦截器也就是CallServerInterceptor进行网络读写操作，那今天的博客就从这里开始。

## 1. 源码解析

讲之前简单介绍下HttpCodec，它是对客户端与服务器进行io操作的封装，默认提供了两个实现Http1Codec、Http2Codec，我们只讲http1.0所以只关注Http1Codec即可

```java
public final class Http1Codec implements HttpCodec {
  final BufferedSource source;
  final BufferedSink sink;
}
```

内部有source和sink两个成员变量用来进行读写操作，这两个流是在ConnectInterceptor建立连接的时候通过socket创建，所以通过他们就可以与远端进行通信了。

```java
	//RealConnection.java
	private void connectSocket(int connectTimeout, int readTimeout, Call call,
      EventListener eventListener) throws IOException {//建立连接
			source = Okio.buffer(Okio.source(rawSocket));
      sink = Okio.buffer(Okio.sink(rawSocket));
  }

  public HttpCodec newCodec(OkHttpClient client, Interceptor.Chain chain,,
      StreamAllocation streamAllocation) throws SocketException {//创建HttpCodec
      socket.setSoTimeout(chain.readTimeoutMillis());
      source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
      return new Http1Codec(client, streamAllocation, source, sink);
  }
```

ok回到主题，对于CallServerInterceptor中的操作，我们大体可以分为四步

1. 写请求头
2. 写请求体
3. 读响应头
4. 读响应体

那源码讲解的时候会将CallServerInterceptor源码做一个拆解，从这四步来分析。

### 1.1 写请求头

```java
	//CallServerInterceptor
	@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();//获取ConnectInterceptor创建的HttpCodec
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();//发起请求的时间

    realChain.eventListener().requestHeadersStart(realChain.call());
    httpCodec.writeRequestHeaders(request);//写请求头
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);
		
    ...
    return response;
  }
```

在CallServerInterceptor中第一步是先拿到HttpCodec，这个类是我们与server进行io操作的封装类，内部是通过Okio与远端进行读写的。

然后调用到httpCodec#writeRequestHeaders(request)进行请求头的写入

```java
	@Override public void writeRequestHeaders(Request request) throws IOException {
    String requestLine = RequestLine.get(
        request, streamAllocation.connection().route().proxy().type());//获取请求行
    writeRequest(request.headers(), requestLine);//写入请求头
  }
```

先获取请求行，然后写入请起头

```java
 	//RequestLine#get
	public static String get(Request request, Proxy.Type proxyType) {
    StringBuilder result = new StringBuilder();
    result.append(request.method());
    result.append(' ');

    if (includeAuthorityInRequestLine(request, proxyType)) {
      result.append(request.url());
    } else {
      result.append(requestPath(request.url()));
    }

    result.append(" HTTP/1.1");
    return result.toString();
  }
```

请求行的获取其实就是method url 协议三段拼接起来的，像这样`POST /app/a/update HTTP/1.1`

```java
  public void writeRequest(Headers headers, String requestLine) throws IOException {
    if (state != STATE_IDLE) throw new IllegalStateException("state: " + state);
    sink.writeUtf8(requestLine).writeUtf8("\r\n");//写入请求行并换行
    for (int i = 0, size = headers.size(); i < size; i++) {//写入请求头
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n");
    }
    sink.writeUtf8("\r\n");//换行
    state = STATE_OPEN_REQUEST_BODY;
  }
```

先写入请求行，然后在遍历写入请求头，最后写完大概像这样

```http
POST /app/a/update HTTP/1.1
Host: XXX.XXX.XXX.XXX:9999
Content-Type: application/x-www-form-urlencoded
```

### 1.2 写请求体

```java
  @Override public Response intercept(Chain chain) throws IOException {
    //...
    
		Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {//如果method不为get和head，并且有请求体
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {//100-continue属于特殊的请求头我们可以忽略
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();//获取请求体长度
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));//创建请求体输出流
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);//写入请求体
        bufferedRequestBody.close();//关闭流
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();//完成请求写入
    
    //...
  }
```

如果非get、head请求并且有请求体，创建输出流对象，写入请求体。

接下来输出流的创建httpCodec#createRequestBody()

```java
	@Override public Sink createRequestBody(Request request, long contentLength) {
    if ("chunked".equalsIgnoreCase(request.header("Transfer-Encoding"))) {//分块传输的则创建newChunkedSink
      return newChunkedSink();
    }

    if (contentLength != -1) {//普通固定长度的则创建newFixedLengthSink
      return newFixedLengthSink(contentLength);
    }

    throw new IllegalStateException(
        "Cannot stream a request body without chunked encoding or a known content length!");
  }

  public Sink newFixedLengthSink(long contentLength) {
    if (state != STATE_OPEN_REQUEST_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_WRITING_REQUEST_BODY;
    return new FixedLengthSink(contentLength);//创建一个FixedLengthSink对象
  }
```

根据当前请求是否需要分块传输做区分，一般情况下都是固定长度的创建一个FixedLengthSink

```java
  private final class FixedLengthSink implements Sink {
    private final ForwardingTimeout timeout = new ForwardingTimeout(sink.timeout());
    private boolean closed;
    private long bytesRemaining;

    FixedLengthSink(long bytesRemaining) {
      this.bytesRemaining = bytesRemaining;//记录要传输的数据量
    }

    @Override public Timeout timeout() {
      return timeout;
    }

    @Override public void write(Buffer source, long byteCount) throws IOException {
      if (closed) throw new IllegalStateException("closed");
      checkOffsetAndCount(source.size(), 0, byteCount);
      if (byteCount > bytesRemaining) {//超出传输数据量抛异常
        throw new ProtocolException("expected " + bytesRemaining
            + " bytes but received " + byteCount);
      }
      sink.write(source, byteCount);
      bytesRemaining -= byteCount;
    }

    @Override public void flush() throws IOException {
      if (closed) return; // Don't throw; this stream might have been closed on the caller's behalf.
      sink.flush();
    }

    @Override public void close() throws IOException {
      if (closed) return;
      closed = true;
      if (bytesRemaining > 0) throw new ProtocolException("unexpected end of stream");
      detachTimeout(timeout);
      state = STATE_READ_RESPONSE_HEADERS;
    }
  }

```

内部其实就是对sink的包装，加上了对长度的判断，如果write的数据长度超过了content-lenght则抛出异常，而`new CountingSink()`也是包装了一层记录了下传输成功的数据量，这里就不细说了。

再看到请求的写入`request.body().writeTo(bufferedRequestBody)`，有多种实现，我们直接看表单的吧

```java
	//FormBody.java
	@Override public void writeTo(BufferedSink sink) throws IOException {
    writeOrCountBytes(sink, false);
  }

 private long writeOrCountBytes(@Nullable BufferedSink sink, boolean countBytes) {
    long byteCount = 0L;

    Buffer buffer;
    if (countBytes) {
      buffer = new Buffer();
    } else {
      buffer = sink.buffer();//获取输出流的buffer
    }

    for (int i = 0, size = encodedNames.size(); i < size; i++) {//将表单数据写入
      if (i > 0) buffer.writeByte('&');
      buffer.writeUtf8(encodedNames.get(i));
      buffer.writeByte('=');
      buffer.writeUtf8(encodedValues.get(i));
    }

    if (countBytes) {
      byteCount = buffer.size();
      buffer.clear();
    }

    return byteCount;
  }
```

for循环写入请求体，最后CallServerInterceptor中调用了`bufferedRequestBody.close()`将请求体写入。

### 1.3 读响应头

```java
  @Override public Response intercept(Chain chain) throws IOException {
    //...
		if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      responseBuilder = httpCodec.readResponseHeaders(false);//获取响应头
    }

    Response response = responseBuilder//创建Response
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
    //...
  }
```

通过httpCodec#readResponseHeaders(false)获取响应头

```java
  @Override public Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
    if (state != STATE_OPEN_REQUEST_BODY && state != STATE_READ_RESPONSE_HEADERS) {
      throw new IllegalStateException("state: " + state);
    }

    try {
      StatusLine statusLine = StatusLine.parse(readHeaderLine());//获取响应中第一行数据解析成StatusLine

      Response.Builder responseBuilder = new Response.Builder()//创建Response.Builder对象
          .protocol(statusLine.protocol)
          .code(statusLine.code)
          .message(statusLine.message)
          .headers(readHeaders());//读取响应头

      if (expectContinue && statusLine.code == HTTP_CONTINUE) {
        return null;
      } else if (statusLine.code == HTTP_CONTINUE) {
        state = STATE_READ_RESPONSE_HEADERS;
        return responseBuilder;
      }

      state = STATE_OPEN_RESPONSE_BODY;
      return responseBuilder;
    } catch (EOFException e) {
      IOException exception = new IOException("unexpected end of stream on " + streamAllocation);
      exception.initCause(e);
      throw exception;
    }
  }
```

读取第一行数据作为响应行

```java
  private String readHeaderLine() throws IOException {
    String line = source.readUtf8LineStrict(headerLimit);
    headerLimit -= line.length();
    return line;
  }
```

随后解析为StatusLine对象，然后在readHeaders()读取响应头

```java
  public Headers readHeaders() throws IOException {
    Headers.Builder headers = new Headers.Builder();
    for (String line; (line = readHeaderLine()).length() != 0; ) {//解析到第一个空行，因为response是用空行来分割响应头和响应体的
      Internal.instance.addLenient(headers, line);//给Headers.Builder添加头部属性
    }
    return headers.build();
  }

    Builder addLenient(String line) {
      int index = line.indexOf(":", 1);
      if (index != -1) {
        return addLenient(line.substring(0, index), line.substring(index + 1));//获取name和value
      } else if (line.startsWith(":")) {
        // Work around empty header names and header names that start with a
        // colon (created by old broken SPDY versions of the response cache).
        return addLenient("", line.substring(1)); // Empty header name.
      } else {
        return addLenient("", line); // No header name.
      }
    }

    Builder addLenient(String name, String value) {//添加到builder中
      namesAndValues.add(name);
      namesAndValues.add(value.trim());
      return this;
    }
```

也是一行一行读，直到读到第一个换行为止，因为响应头和响应体之间是通过一个空行来区分。然后创建Header对象传入responseBuilder，最后构建出response。

### 1.4 读响应体

```java
	@Override public Response intercept(Chain chain) throws IOException {
    //...
		response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))//读取响应体
          .build();
    //...
		}
```

直接看到httpCodec.openResponseBody(response)

```java
  @Override public ResponseBody openResponseBody(Response response) throws IOException {
    streamAllocation.eventListener.responseBodyStart(streamAllocation.call);
    String contentType = response.header("Content-Type");

    if (!HttpHeaders.hasBody(response)) {//一般情况下都是走到这个分支
      Source source = newFixedLengthSource(0);
      return new RealResponseBody(contentType, 0, Okio.buffer(source));
    }

    if ("chunked".equalsIgnoreCase(response.header("Transfer-Encoding"))) {
      Source source = newChunkedSource(response.request().url());
      return new RealResponseBody(contentType, -1L, Okio.buffer(source));
    }

    long contentLength = HttpHeaders.contentLength(response);
    if (contentLength != -1) {
      Source source = newFixedLengthSource(contentLength);
      return new RealResponseBody(contentType, contentLength, Okio.buffer(source));
    }

    return new RealResponseBody(contentType, -1L, Okio.buffer(newUnknownLengthSource()));
  }

```

根据不同类型创建不同的流去读请求体数据，一般情况走的一个分支，newFixedLengthSource(0)，跟前面创建Sink差不多的流程，这里是包装了一下source加了点功能，然后传入到了RealResponseBody对象中，这里body对象就创建好了，那么Response也就创建好了。

```java
public final class RealResponseBody extends ResponseBody {
  private final @Nullable String contentTypeString;
  private final long contentLength;
  private final BufferedSource source;

  public RealResponseBody(
      @Nullable String contentTypeString, long contentLength, BufferedSource source) {
    this.contentTypeString = contentTypeString;
    this.contentLength = contentLength;
    this.source = source;
  }

  @Override public MediaType contentType() {
    return contentTypeString != null ? MediaType.parse(contentTypeString) : null;
  }

  @Override public long contentLength() {
    return contentLength;
  }

  @Override public BufferedSource source() {
    return source;
  }
}

```

而传入的source则通过成员变量存储了起来，直到我们调用ResponseBody#string()方法的时候才将响应体数据读出

```java
  public final String string() throws IOException {
    BufferedSource source = source();
    try {
      Charset charset = Util.bomAwareCharset(source, charset());
      return source.readString(charset);//从流中读取数据
    } finally {
      Util.closeQuietly(source);//关闭流
    }
  }
```

实际也就是从source中读取数据，并且读完后它调用了source#close将流关闭了，所以当我们多次执行ResponseBody#string()的时候会抛出异常因为流已经关闭了。

## 2. 总结

CallServerInterceptor内部就是通过ConnectInterceptor创建的HttpCodec与远端进行io操作，流程上就是四步

1. 写请求头
2. 写请求体
3. 读响应头
4. 读响应体

也没啥可说的了就酱，okhttp全部拦截器终于看完了(＾－＾)V
