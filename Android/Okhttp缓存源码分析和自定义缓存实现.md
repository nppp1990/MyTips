





[TOC]

### 缓存的一般思路

下面是我理解的网络请求框架的缓存基本实现。大致的过程是有缓存用缓存的数据，没缓存发起http请求取数据，得到最新数据后存到缓存里。

```flow
st=>start: 开始
cache?=>condition: 缓存可用?
getCache=>operation: 从缓存获取数据
request=>operation: 从网络请求数据
success?=>condition: 获取数据成功
updateCache=>operation: 更新缓存
show=>operation: show
e=>end: 结束

st->cache?
cache?(yes)->getCache->show
cache?(no)->request->success?(yes)->updateCache->show->e
```

那么Okhttp怎么实现缓存的，我们从Okhttp发起一次请求的全过程中来看缓存是怎么实现的

### Okhttp请求过程源码分析

最简单的使用（以下代码都是okhttp3.8.0为基础）：

~~~java
Response response = client.newCall(request).execute()
~~~

追踪到Call接口的实现类RealCall的方法execute

```
@Override public Response execute() throws IOException {
  synchronized (this) {
    // 判断是否在执行，是则抛出异常
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  // 初始化跟踪stack trace的对象，用来做日志，所以可以忽略先
  captureCallStackTrace();
  try {
    //将异步的请求丢到异步的双端队列(Deque<RealCall> runningSyncCalls)中等待处理，这里可以先忽略，直接看同步的结果
    client.dispatcher().executed(this);
    //获取Response
    Response result = getResponseWithInterceptorChain();
    if (result == null) throw new IOException("Canceled");
    return result;
  } finally {
    client.dispatcher().finished(this);
  }
}
```

很明显，获取response在getResponseWithInterceptorChain这个方法里。这里代码很简单，就是初始化一个interceptor列表，然后调用**RealInterceptorChain**的**proceed**函数。

~~~java
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

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
~~~

#### interceptors（拦截器）

那么这些interceptor有什么用呢，我们挑几个重要的一一看一下，**记住这些interceptors的add顺序很重要**

client.interceptors()：

依次追踪到interceptors的赋值的地方

```
public Builder addInterceptor(Interceptor interceptor) {
  interceptors.add(interceptor);
  return this;
}
```

```
public List<Interceptor> interceptors() {
  return interceptors;
}
```

这个是不是很熟悉，这个就是我们利用OkHttpClient.Builder builder构造okhttpClient的地方传入的interceptor，也就是常说的application interceptor

CacheInterceptor：看名字很明显是用来做缓存的

ConnectInterceptor：用来建立http连接

client.networkInterceptors()：同client.interceptors()，是我们创建okhttpclient时传入的networkInterceptor

CallServerInterceptor：向server发请求的

#### RealInterceptorChain.proceed(request)

追踪到下面的方法，这时候传入的streamAllocation，httpCodec，connection都是null，index=0

~~~java
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    // 判断index是否越界
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    // 创建一个新的RealInterceptorChain，除了index+1，其他的参数都和上一个RealInterceptorChain保持不变
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    // 获取当前index的Interceptor
    Interceptor interceptor = interceptors.get(index);
    // 执行当前Interceptor的intercept方法，传入的参数为下一个RealInterceptorChain
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
  }
~~~

所以RealInterceptorChain.proceed的大致过程如下

1. **获取下一个RealInterceptorChain next**
2. **调用当前的interceptor的intercept方法，传入参数为next**

所以我们要追踪interceptor的intercept方法,下面我以我项目里的一个做统计的intercptor为例来分析

~~~java
public class NetStatisticsInterceptor implements Interceptor {

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        HttpUrl httpUrl = request.url();
        HttpUrl.Builder urlBuilder = httpUrl.newBuilder();
        if (httpUrl.queryParameter("app_version") == null) {
            urlBuilder.addQueryParameter("app_version", BaseConfig.versionName);
        }
        // 用chain.request()构造一个新的传入统计参数的request，作为参数调用chain.proceed
        return chain.proceed(request.newBuilder().url(urlBuilder.build()).build());
    }
}
~~~

这里对旧的oldRequest做了一堆处理，加入了一些通用的统计参数，包装成生成了一个新的newRequest，然后调用chain.proceed方法，这里又会重新调用RealInterceptorChain.proceed的方法，只是参数index+1了，request为重新包装后的request了(其他的参数也可能变了，取决于Interceptor怎么写)。接着又会走到RealInterceptorChain.proceed代码里，走下一个Interceptor的流程。

可以得出如下结论：

1. 只要Interceptor的intercept方法调用了chain.proceed(request),就会调用Interceptor列表里的下一个Interceptor；反之可以不调用chain.proceed来打断这个请求链
2. 我们自定义的application interceptor和network interceptor时，都必须返回chain.proceed得到的结果；否则就会打断okhttp内部的请求链
3. 写application interceptor时，**在调用chain.proceed(request)之前包装request**
4. 写network interceptor时,**在调用chain.proceed(request)之后得到的response包装response**

看到这里的代码设计，是不是和**[职责链模式](https://zh.wikipedia.org/wiki/%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F)**很相似，唯一不同的是okhttp利用index自增的方式来实现每个拦截器的传递。这里我必须感叹下，代码设计的真的很巧妙，还有就是设计模式这东西平常看不出有啥用，到实际碰到了真的很棒。

了解完这些拦截器怎么运行的，接下来具体看看各个拦截器是怎么把请求给串联起来的。

#### 应用拦截器（client.interceptors())

这个我们常说的application interceptor因为在拦截器list的最前面，所以最先执行，一般用于给request做一些简单的包装，例如添加参数，修改header等

#### CacheInterceptor

直接看intercept方法

~~~java
  @Override public Response intercept(Chain chain) throws IOException {
    // 根据url获取本地缓存
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();

    // 用当前时间now、当前的请求request、本地缓存cacheCandidate来构造CacheStrategy对象
    // 调用strategy对象的get方法去判断本地缓存cacheCandidate是否可用
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
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

    // If we don't need the network, we're done.如果networkRequest为null就表示走本地缓存
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    // 走后面的interceptor链去取网络数据得到networkResponse
    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    // 用networkResponse、cacheResponse构造新的response
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // 如果response符合缓存的策略需要缓存，则put到cache中
        // Offer this request to the cache.
        // 这里追踪到put中，可以发现只有method为GET才会add到cache中，所以okhttp是只支持get请求的缓存的；且key为response.request().url()
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
~~~

大致流程如下

1. 获取本地缓存cacheCandidate
2. 如果本地缓存可用则直接返回cacheCandidate，从而打断interceptor链
3. 走剩下的interceptor获取networkResponse
4. networkResponse、cacheResponse构造新的response
5. 根据新的response里的header定制缓存策略，存入缓存中

~~~flow
st=>start: 开始
getCache=>operation: 获取本地缓存cache
newStrategy=>operation: cache+time+request构造CacheStrategy对象
cache?=>condition: CacheStrategy判断缓存可用?
runChain=>operation: 走剩下的interceptor，获取networkResponse，并且生成最终的response
needUpdateCache?=>condition: 是否需要更新缓存
updateCache=>operation: 更新缓存

show=>operation: show
e=>end: 结束，返回response
e2=>end: 结束，返回cacheResponse

st->getCache->newStrategy->cache?
cache?(no)->runChain->needUpdateCache?
needUpdateCache?(yes)->updateCache->e
needUpdateCache?(no)->e
cache?(yes)->e2
~~~

##### CacheStrategy

从上面的代码来看，主要的缓存策略都是在这个类里实现。

我们关注这两个变量,networkRequest为null就不走网络取数据，cacheResponse为null则不用缓存

~~~java
  /** The request to send on the network, or null if this call doesn't use the network. */
  public final @Nullable Request networkRequest;

  /** The cached response to return or validate; or null if this call doesn't use a cache. */
  public final @Nullable Response cacheResponse;
~~~

~~~java
    public CacheStrategy get() {
      CacheStrategy candidate = getCandidate();

      if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
        // We're forbidden from using the network and the cache is insufficient.
        return new CacheStrategy(null, null);
      }

      return candidate;
    }
~~~

追踪到public CacheStrategy get()方法

~~~java
    private CacheStrategy getCandidate() {
      // No cached response.
      if (cacheResponse == null) {
        return new CacheStrategy(request, null);
      }

      // Drop the cached response if it's missing a required handshake.
      // 请求为https且缓存没有TLS握手
      if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
      }

      // If this response shouldn't have been stored, it should never be used
      // as a response source. This check should be redundant as long as the
      // persistence store is well-behaved and the rules are constant.
      // 跟进缓存Response的code，response和request的cache-control的noStore字段判断是否需要缓存
      if (!isCacheable(cacheResponse, request)) {
        return new CacheStrategy(request, null);
      }

      CacheControl requestCaching = request.cacheControl();
      // 请求的header不要缓存
      if (requestCaching.noCache() || hasConditions(request)) {
        return new CacheStrategy(request, null);
      }

      long ageMillis = cacheResponseAge();
      long freshMillis = computeFreshnessLifetime();

      if (requestCaching.maxAgeSeconds() != -1) {
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
      }

      long minFreshMillis = 0;
      if (requestCaching.minFreshSeconds() != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
      }

      long maxStaleMillis = 0;
      CacheControl responseCaching = cacheResponse.cacheControl();
      if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
      }

      if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        Response.Builder builder = cacheResponse.newBuilder();
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
        return new CacheStrategy(null, builder.build());
      }

      // Find a condition to add to the request. If the condition is satisfied, the response body
      // will not be transmitted.
      String conditionName;
      String conditionValue;
      if (etag != null) {
        conditionName = "If-None-Match";
        conditionValue = etag;
      } else if (lastModified != null) {
        conditionName = "If-Modified-Since";
        conditionValue = lastModifiedString;
      } else if (servedDate != null) {
        conditionName = "If-Modified-Since";
        conditionValue = servedDateString;
      } else {
        return new CacheStrategy(request, null); // No condition! Make a regular request.
      }

      Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
      Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

      Request conditionalRequest = request.newBuilder()
          .headers(conditionalRequestHeaders.build())
          .build();
      return new CacheStrategy(conditionalRequest, cacheResponse);
    }
~~~

看这个方法大部分都是返回CacheStrategy(request, null)也就是走网络，那么我们直接看唯一的返回缓存的代码：return new CacheStrategy(null, builder.build());什么条件呢？

~~~java
// ageMillis是response的maxAge时间和当前时间算出来的cache的有效时间。。。。具体我也没看明白哈
//response不是no-cache且(ageMillis+request的min-fresh时间)<(request的max-age时间+request的max-stale)
!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis
~~~

总结下

1. request的header有only-if-cached：啥缓存都不用
2. 没有缓存：当然不用缓存
3. request为https且缓存丢失了TLS握手：不用缓存
4. request或者response的header有no-store：不用缓存
5. response除了200一些的status code以外：不用缓存
6. **满足这个条件ageMillis + minFreshMillis < freshMillis + maxStaleMillis的request和response：用缓存**
7. 其他的一些情况（我看晕了）
8. **总之就是根据request和response的header的cache-control来做缓存，我们可以严格按照http协议的[Cache-Control](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control)来做缓存策略**，而不用去看okhttp协议怎么实现的（嗯，okhttp应该是严格按照http协议来写的吧？）

#### ConnectInterceptor,CallServerInterceptor

##### ConnectInterceptor

Opens a connection to the target server and proceeds to the next interceptor

关键类：StreamAllocation

##### CallServerInterceptor

This is the last interceptor in the chain. It makes a network call to the server.

### 最佳实践

#### 服务端控制缓存

1. 客户端请求时，header传入想要的缓存时间策略，例如

   ~~~java
   @Headers("Cache-Control: no-cache")// 不要缓存
   @Headers("Cache-Control: public, max-age=604800")//缓存时间为604800秒
   ~~~

2. 服务端指定缓存策略，返回相应的response Cache-Control

然而很不幸，大部分的服务端都没有返回Cache-Control来控制缓存，所以就有了下面的办法

#### 客户端控制缓存时间

1. 客户端传入header

   ~~~java
   @Headers("Cache-Control: public, max-age=30")//缓存时间为30秒
   ~~~

2. 添加networkInterceptor

   ~~~java
       @Override
       public Response intercept(Chain chain) throws IOException {
           Request request = chain.request();
           Response originalResponse = chain.proceed(request);
           if (TextUtils.isEmpty(originalResponse.header("Cache-Control"))) {
               // 这里把request传入的header传递给response
               return originalResponse.newBuilder().header("Cache-Control", request.header("Cache-Control"))
                       .build();
           }
           return originalResponse;
       }
   ~~~

#### 客户端控制缓存时间，同时要求无网络的时候使用缓存

有网络的时候同上；无网络的时候，如果超过一天则显示error，没超过一天用缓存

1. 客户端传入header同上

2. networkInterceptor同上

3. 添加applicationInterceptor,传入一个max-age为无限大数的header就能强制用缓存了，或者

   ~~~java
       @Override
       public Response intercept(Chain chain) throws IOException {
           Request request = chain.request();
           CacheControl cacheControl = request.cacheControl();
           boolean noCache = cacheControl.noCache() || cacheControl.noStore() || cacheControl.maxAgeSeconds() == 0;
           // 如果header强制要求不用缓存就不走这个逻辑
           if (!noCache && !NetworkUtils.isNetworkAvailable(context)) {
               Request.Builder builder = request.newBuilder();
               //if network not available, load in cache
               CacheControl newCacheControl = new CacheControl.Builder()
                       .maxStale(60, TimeUnit.SECONDS).build();
               request = builder.cacheControl(newCacheControl).build();
               return chain.proceed(request);
           }
           return chain.proceed(request);
       }
   ~~~

#### 客户端控制缓存时间，同时要求无网络的时候使用缓存，并且这个缓存超过一天就失效了

基本同上，但是传入的header不是maxAge而是max-stale，设置缓存过期后还能可用的时间为一天即可

~~~java
            CacheControl newCacheControl = new CacheControl.Builder()
                    .maxStale(ONE_DAY, TimeUnit.SECONDS).build();
~~~

#### 客户端控制缓存时间,request的时间和response的时间不同

前面的几个方式，response的header实际上是从request取出来的，也就是说我们的response的header时间和request的header时间是一样的。但是如果要不一样的情况怎么办呢？举个我项目里的例子

1. 发送A请求（缓存为30分钟），发现这个商品没有买
2. 花钱把这个商品买了，再次请求A请求刷新页面
3. 因为A请求有30分钟缓存没有刷新数据；于是乎我修改了request的header为不使用缓存(也就是age为0),这时数据刷新了
4. 几分钟后，我下次进来这个页面，再次请求A（因为之前age为0，所以并没有缓存），我又发了次请求（实际我期望的是使用缓存的）

实际上我希望的是在步骤3里发送A请求时，request的header为age=0，response的age=30min，那么怎么实现呢，所以提供了下面的方法

首先提供了一个工具类，用来存放header的时间和生成header。这里用ThreadLocal变量存放了response的时间

~~~java
public final class NetAccessStrategy {
    private NetAccessStrategy() {

    }
    private static final ThreadLocal<Integer> localCacheTime = new ThreadLocal<>();

    public static void setThreadLocalCacheTime(int cacheTime) {
        localCacheTime.set(cacheTime);
    }

    public static int getThreadLocalCacheTime() {
        Integer time = localCacheTime.get();
        localCacheTime.remove();
        if (time == null) {
            return 0;
        }
        return time;
    }

    public static final String NET_REQUEST = "net-";


    /**
     * @param requestCacheTime 本地缓存在超过这个时间后失效
     * @param localCacheTime   本地缓存的时间
     * @return
     */
    public static String getRequestNetHeader(int requestCacheTime, int localCacheTime) {
        return NET_REQUEST + requestCacheTime + "-" + +localCacheTime;
    }

    public static int[] getRequestCacheTime(String netHeader) {
        int index1 = netHeader.indexOf("-", 1);
        int index2 = netHeader.indexOf("-", index1 + 1);
        int time1 = -1;
        int time2 = -1;
        if (index1 != -1 && index2 != -1) {
            try {
                time1 = Integer.parseInt(netHeader.substring(index1 + 1, index2));
            } catch (NumberFormatException ignored) {
            }
            try {
                time2 = Integer.parseInt(netHeader.substring(index2 + 1));
            } catch (NumberFormatException ignored) {
            }
        }
        return new int[]{time1, time2};
    }
}
~~~

在application Interceptor里加上

~~~java
// 如果发现net-开头的自定义header时
if (header.startsWith(NetAccessStrategy.NET_REQUEST)) {
            Request.Builder builder = request.newBuilder();
            // 解析得到request和response的时间
            int[] timeArray = NetAccessStrategy.getRequestCacheTime(header);
            // 传入request的age时间
            CacheControl cacheControl = new CacheControl.Builder().maxAge(timeArray[0], TimeUnit.SECONDS).build();
            // 存入response的时间
            NetAccessStrategy.setThreadLocalCacheTime(timeArray[1]);
            builder.cacheControl(cacheControl);
            return chain.proceed(builder.build());
        }
~~~

在network Interceptor里加上

~~~java
 Response originalResponse = chain.proceed(request);
            int time = NetAccessStrategy.getThreadLocalCacheTime();
            if (time > 0) {
                // 取出response time，如果大于0则放到header里
                return originalResponse.newBuilder().header("Cache-Control", "public, max-age=" + time)
                        .build();
            }
            return originalResponse.newBuilder().header("Cache-Control", request.header("Cache-Control"))
                    .build();
~~~

发请求时加上header，这样就能实现强制刷新，且缓存为300秒的功能了

~~~java
NetAccessStrategy.getRequestNetHeader(0, 300)
~~~

