---
title: Volley 知识梳理 - Volley解析
date: 2017-02-22 00:25
categories : Volley 知识梳理
---
# 一、网络请求核心
# 1.1 `Network`
`Volley`与网络请求相关的接口有两个： 
```
/**
 * An interface for performing requests.
 */
public interface Network {
    /**
     * Performs the specified request.
     * @param request Request to process
     * @return A {@link NetworkResponse} with data and caching metadata; will never be null
     * @throws VolleyError on errors
     */
    public NetworkResponse performRequest(Request<?> request) throws VolleyError;
}
```
## 1.2 `HttpStack`
```
public interface HttpStack {
    /**
     * Performs an HTTP request with the given parameters.
     *
     * <p>A GET request is sent if request.getPostBody() == null. A POST request is sent otherwise,
     * and the Content-Type header is set to request.getPostBodyContentType().</p>
     *
     * @param request the request to perform
     * @param additionalHeaders additional headers to be sent together with
     *         {@link Request#getHeaders()}
     * @return the HTTP response
     */
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError;
}
```
## 1.3 具体实现
对于上面的两个接口，它们各有对应的实现类：
- `Network`：`BasicNetwork`
- `HttpStack`：在`SDK`大于等于九时，其实现类是`HurlStack`，小于九时，对应的实现类是`HttpClientStack`。

> `BasicNetwork`和`HttpStack`的两个实现类的关系是：`HttpStack`是`BasicNetwork`的一个成员变量，当`Volley`传入`Request`，调用`BasicNetwork#performRequest`后，在它的内部实际是通过`HttpStack#performRequest`发起网络请求，并把标准网络请求返回结果`HttpResponse`封装成`Volley`的`NetworkResponse`。

在看具体的请求之前，我们需要先了解一下和请求相关的两个类`Request`和`NetworkResponse`。
### 1.3.1 `Request`
```
public abstract class Request<T> implements Comparable<Request<T>> {

    /**
     * Default encoding for POST or PUT parameters. See {@link #getParamsEncoding()}.
     */
    private static final String DEFAULT_PARAMS_ENCODING = "UTF-8";

    /**
     * Supported request methods.
     */
    public interface Method {
        int DEPRECATED_GET_OR_POST = -1;
        int GET = 0;
        int POST = 1;
        int PUT = 2;
        int DELETE = 3;
        int HEAD = 4;
        int OPTIONS = 5;
        int TRACE = 6;
        int PATCH = 7;
    }

    /** An event log tracing the lifetime of this request; for debugging. */
    private final MarkerLog mEventLog = MarkerLog.ENABLED ? new MarkerLog() : null;

    /**
     * Request method of this request.  Currently supports GET, POST, PUT, DELETE, HEAD, OPTIONS,
     * TRACE, and PATCH.
     */
    private final int mMethod;

    /** URL of this request. */
    private final String mUrl;

    /** Default tag for {@link TrafficStats}. */
    private final int mDefaultTrafficStatsTag;

    /** Listener interface for errors. */
    private final Response.ErrorListener mErrorListener;

    /** Sequence number of this request, used to enforce FIFO ordering. */
    private Integer mSequence;

    /** The request queue this request is associated with. */
    private RequestQueue mRequestQueue;

    /** Whether or not responses to this request should be cached. */
    private boolean mShouldCache = true;

    /** Whether or not this request has been canceled. */
    private boolean mCanceled = false;

    /** Whether or not a response has been delivered for this request yet. */
    private boolean mResponseDelivered = false;

    // A cheap variant of request tracing used to dump slow requests.
    private long mRequestBirthTime = 0;

    /** Threshold at which we should log the request (even when debug logging is not enabled). */
    private static final long SLOW_REQUEST_THRESHOLD_MS = 3000;

    /** The retry policy for this request. */
    private RetryPolicy mRetryPolicy;

    /**
     * When a request can be retrieved from cache but must be refreshed from
     * the network, the cache entry will be stored here so that in the event of
     * a "Not Modified" response, we can be sure it hasn't been evicted from cache.
     */
    private Cache.Entry mCacheEntry = null;

    /** An opaque token tagging this request; used for bulk cancellation. */
    private Object mTag;

}
```
`Request<T>`实现了`Comparable<Request<T>>`接口，它是所有网络请求的基类，实现这个接口是为了比较各个请求之间的优先级。
它定义了内部接口`Method`，表示支持的方法：
```
public interface Method {
    int DEPRECATED_GET_OR_POST = -1;
    int GET = 0;
    int POST = 1;
    int PUT = 2;
    int DELETE = 3;
    int HEAD = 4;
    int OPTIONS = 5;
    int TRACE = 6;
    int PATCH = 7;
}
```

除此之外还定义了成员变量包括：
- `mMethod`：该请求所对应的方法。
- `mUrl`：请求的`Url`。
- `mSequence`：序列号。
- `mRequestQueue`：该请求所加入的队列。
- `mShouldCache`：是否需要缓存。
- `mCanceled`：该请求是否已经取消。
- `mResponseDelivered`：该请求是否已经`delivered`。
- `mRequestBirthTime`：请求开始的时间。
- `RetryPolicy mRetryPolicy`：重试策略。
- `Cache.Entry mCacheEntry`：当一个请求结果可以从缓存中获得，但必须通过网络请求来刷新，缓存就可以存放在这里，当出现`Not Modified`时，将它返回。
- `mTag`：标识，用来一次性取消多个请求。

它有两个抽象方法：
```
    /**
     * Subclasses must implement this to parse the raw network response
     * and return an appropriate response type. This method will be
     * called from a worker thread.  The response will not be delivered
     * if you return null.
     * @param response Response from the network
     * @return The parsed response, or null in the case of an error
     */
    abstract protected Response<T> parseNetworkResponse(NetworkResponse response);

    /**
     * Subclasses must implement this to perform delivery of the parsed
     * response to their listeners.  The given response is guaranteed to
     * be non-null; responses that fail to parse are not delivered.
     * @param response The parsed response returned by
     * {@link #parseNetworkResponse(NetworkResponse)}
     */
    abstract protected void deliverResponse(T response);
```
## 1.3.2 `NetworkResponse`
`NetworkResponse`是`BasicNetwork#performRequest`返回的，它仅仅包含返回的`content`和`Header`，这样就便于后面对`Response`进行转换。
```
public class NetworkResponse {
    /**
     * Creates a new network response.
     * @param statusCode the HTTP status code
     * @param data Response body
     * @param headers Headers returned with this response, or null for none
     * @param notModified True if the server returned a 304 and the data was already in cache
     * @param networkTimeMs Round-trip network time to receive network response
     */
    public NetworkResponse(int statusCode, byte[] data, Map<String, String> headers,
            boolean notModified, long networkTimeMs) {
        this.statusCode = statusCode;
        this.data = data;
        this.headers = headers;
        this.notModified = notModified;
        this.networkTimeMs = networkTimeMs;
    }

    public NetworkResponse(int statusCode, byte[] data, Map<String, String> headers,
            boolean notModified) {
        this(statusCode, data, headers, notModified, 0);
    }

    public NetworkResponse(byte[] data) {
        this(HttpStatus.SC_OK, data, Collections.<String, String>emptyMap(), false, 0);
    }

    public NetworkResponse(byte[] data, Map<String, String> headers) {
        this(HttpStatus.SC_OK, data, headers, false, 0);
    }

    /** The HTTP status code. */
    public final int statusCode;

    /** Raw data from this response. */
    public final byte[] data;

    /** Response headers. */
    public final Map<String, String> headers;

    /** True if the server returned a 304 (Not Modified). */
    public final boolean notModified;

    /** Network roundtrip time in milliseconds. */
    public final long networkTimeMs;
}
```
它包含以下成员变量：
- `statusCode`：`HTTP`状态码。
- `byte[] data`：返回数据。
- `Map<String, String> headers`：返回的头部。
- `boolean notModified`：如果服务器返回了`304`，那么为`true`。
- `long networkTimesMs`：网络请求所消耗的时间。

### 1.3.3 下面我们来看一下`BasicNetwork`的实现代码：
```
@Override
    public NetworkResponse performRequest(Request<?> request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
        while (true) {
            HttpResponse httpResponse = null;
            byte[] responseContents = null;
            Map<String, String> responseHeaders = Collections.emptyMap();
            try {
                // Gather headers.
                Map<String, String> headers = new HashMap<String, String>();
                addCacheHeaders(headers, request.getCacheEntry());
                httpResponse = mHttpStack.performRequest(request, headers);
                StatusLine statusLine = httpResponse.getStatusLine();
                int statusCode = statusLine.getStatusCode();

                responseHeaders = convertHeaders(httpResponse.getAllHeaders());
                // Handle cache validation.
                if (statusCode == HttpStatus.SC_NOT_MODIFIED) {

                    Entry entry = request.getCacheEntry();
                    if (entry == null) {
                        return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,
                                responseHeaders, true,
                                SystemClock.elapsedRealtime() - requestStart);
                    }

                    // A HTTP 304 response does not have all header fields. We
                    // have to use the header fields from the cache entry plus
                    // the new ones from the response.
                    // http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3.5
                    entry.responseHeaders.putAll(responseHeaders);
                    return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data,
                            entry.responseHeaders, true,
                            SystemClock.elapsedRealtime() - requestStart);
                }

                // Some responses such as 204s do not have content.  We must check.
                if (httpResponse.getEntity() != null) {
                  responseContents = entityToBytes(httpResponse.getEntity());
                } else {
                  // Add 0 byte response as a way of honestly representing a
                  // no-content request.
                  responseContents = new byte[0];
                }

                // if the request is slow, log it.
                long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
                logSlowRequests(requestLifetime, request, responseContents, statusLine);

                if (statusCode < 200 || statusCode > 299) {
                    throw new IOException();
                }
                return new NetworkResponse(statusCode, responseContents, responseHeaders, false,
                        SystemClock.elapsedRealtime() - requestStart);
            } catch (SocketTimeoutException e) {
                attemptRetryOnException("socket", request, new TimeoutError());
            } catch (ConnectTimeoutException e) {
                attemptRetryOnException("connection", request, new TimeoutError());
            } catch (MalformedURLException e) {
                throw new RuntimeException("Bad URL " + request.getUrl(), e);
            } catch (IOException e) {
                int statusCode = 0;
                NetworkResponse networkResponse = null;
                if (httpResponse != null) {
                    statusCode = httpResponse.getStatusLine().getStatusCode();
                } else {
                    throw new NoConnectionError(e);
                }
                VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
                if (responseContents != null) {
                    networkResponse = new NetworkResponse(statusCode, responseContents,
                            responseHeaders, false, SystemClock.elapsedRealtime() - requestStart);
                    if (statusCode == HttpStatus.SC_UNAUTHORIZED ||
                            statusCode == HttpStatus.SC_FORBIDDEN) {
                        attemptRetryOnException("auth",
                                request, new AuthFailureError(networkResponse));
                    } else {
                        // TODO: Only throw ServerError for 5xx status codes.
                        throw new ServerError(networkResponse);
                    }
                } else {
                    throw new NetworkError(networkResponse);
                }
            }
        }
    }
```
上面的代码对应的具体流程如下：
- 记录请求开始时间
- 通过`Request#mCacheEntry`来构建新的请求的`Header`
- 把`Request`和`Header`传给`HttpStack`
- `HttpStack`发起请求之后，获得`HttpResponse`的`StatusCode`和`Header`
- 根据`StatusCode`来构建`NetworkResponse`，判断是否是`304`：
 - 是，那么再判断`mCacheEntry`是否为空，如果为空，那么仅仅传入头部构建；否则更新`mCacheEntry`的头部，传入`mCacheEntry.data`来构建。
 - 否，从`Entity`中获得`byte[]`构建`NetworkResponse`。
- 如果在请求或者处理过程当中，发生了异常，那么会根据情况，返回不同的异常，最终都是一个`VolleyError`，它的子类包括：
 - `NetworkError`：用来表示这是在发起请求过程中所产生的错误。
 - `ParseError`：用来表示服务器的数据不能被解析。
 - `TimeoutError`：用来表示`socket`连接超时。
 - `ServerError`：用来表示服务器返回了一个异常响应。
 - `NoConnectionError`：表示没有可建立的连接。

## 1.3.4 `RetryPolicy`
在上面的请求过程当中，我们看到当发生`SocketTimeoutException/ConnectTimeoutException`，会调用
`attemptRetryOnException`，而在`Request`的构造函数中，会给它传入一个`DefaultRetryPolicy`：
```
    //BasicNetwork.java
    /**
     * Attempts to prepare the request for a retry. If there are no more attempts remaining in the
     * request's retry policy, a timeout exception is thrown.
     * @param request The request to use.
     */
    private static void attemptRetryOnException(String logPrefix, Request<?> request,
            VolleyError exception) throws VolleyError {
        RetryPolicy retryPolicy = request.getRetryPolicy();
        int oldTimeout = request.getTimeoutMs();

        try {
            retryPolicy.retry(exception);
        } catch (VolleyError e) {
            request.addMarker(
                    String.format("%s-timeout-giveup [timeout=%s]", logPrefix, oldTimeout));
            throw e;
        }
        request.addMarker(String.format("%s-retry [timeout=%s]", logPrefix, oldTimeout));
    }

    //DefaultRetryPolicy.java
        /**
     * Prepares for the next retry by applying a backoff to the timeout.
     * @param error The error code of the last attempt.
     */
    @Override
    public void retry(VolleyError error) throws VolleyError {
        mCurrentRetryCount++;
        mCurrentTimeoutMs += (mCurrentTimeoutMs * mBackoffMultiplier);
        if (!hasAttemptRemaining()) {
            throw error;
        }
    }
```
我们可以看到，它其实是修改当前`Request`对应的`RetryPolicy`中的超时时间，这样再下次请求时，该`Request`所允许超时的时间就会变长，从而减少超时情况的发生。
## 1.4 小结
在外界看来，`BasicNetwork`就是接受`Request`，返回`NetworkResponse`，在执行过程当中，有可能抛出`VolleyError`。

# 二、`NetworkResponse`转换为`Response`
在上面的例子当中，我们看到`Request`必须要实现一个抽象方法，将网络请求的返回结果`NetworkResponse`转换成为`Response`，来递交给下一级，它包含以下几个成员变量：
```
    /** Parsed response, or null in the case of error. */
    public final T result;

    /** Cache metadata for this response, or null in the case of error. */
    public final Cache.Entry cacheEntry;

    /** Detailed error information if errorCode != OK */
    public final VolleyError error;
```
下面是`StringRequest`的实现：
```
@Override
protected Response<String> parseNetworkResponse(NetworkResponse response) {
    String parsed;
    try {
        parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
    } catch (UnsupportedEncodingException e) {
        parsed = new String(response.data);
    }
    return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
}
```
在`NetworkDispatcher`中，当`Request`解析完请求后，会把`cacheEntry`保存起来，而它的`Key`来自于`Request#getCacheKey`：
```
if (request.shouldCache() && response.cacheEntry != null) {
    mCache.put(request.getCacheKey(), response.cacheEntry);
    request.addMarker("network-cache-written");
}
```
这样就把`result`和`cache`分开来了。
# 三、缓存相关`Cache`
- `Cache`是一个接口，它定义了一个**缓存管理类**应当实现的方法，在`Volley`当中，我们默认实现的缓存管理类是`DiskBasedCache`，在后面的分析中，它分别被传给了`NetworkDispatcher`和`CacheDispatcher`，在前者当中写入缓存，在后者当中读出缓存。
- `Cache`中有一个内部类`Entry`，它定义缓存的数据结构，当使用者希望缓存请求结果的时候，那么就需要构建一个`Cache.Entry`，然后在构建`Response`的时候将它传入，这样在`NetworkDispatcher`中就可以写入到持久性存储当中。
```
  /**
     * Data and metadata for an entry returned by the cache.
     */
    public static class Entry {
        /** The data returned from cache. */
        public byte[] data;

        /** ETag for cache coherency. */
        public String etag;

        /** Date of this response as reported by the server. */
        public long serverDate;

        /** The last modified date for the requested object. */
        public long lastModified;

        /** TTL for this record. */
        public long ttl;

        /** Soft TTL for this record. */
        public long softTtl;

        /** Immutable response headers as received from server; must be non-null. */
        public Map<String, String> responseHeaders = Collections.emptyMap();

        /** True if the entry is expired. */
        public boolean isExpired() {
            return this.ttl < System.currentTimeMillis();
        }

        /** True if a refresh is needed from the original data source. */
        public boolean refreshNeeded() {
            return this.softTtl < System.currentTimeMillis();
        }
    }
```

# 四、`NetworkDispatcher`和`CacheDispatcher`
### 4.1 `CacheDispatcher`
它是一个线程，它的构造函数中传入了：
- `cacheQueue`：缓存的处理队列
- `networkQueue`：网络的处理队列
- `cache`：缓存管理类
- `delivery`：递送类。

上面这两个队列的类型为无界队列`BlockingQueue<Request<?>>`，我们来看一下，在其`run()`方法中的具体操作：
```
    @Override
    public void run() {
        if (DEBUG) VolleyLog.v("start new dispatcher");
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        // Make a blocking call to initialize the cache.
        mCache.initialize();

        while (true) {
            try {
                // Get a request from the cache triage queue, blocking until
                // at least one is available.
                final Request<?> request = mCacheQueue.take();
                request.addMarker("cache-queue-take");

                // If the request has been canceled, don't bother dispatching it.
                if (request.isCanceled()) {
                    request.finish("cache-discard-canceled");
                    continue;
                }

                // Attempt to retrieve this item from cache.
                Cache.Entry entry = mCache.get(request.getCacheKey());
                if (entry == null) {
                    request.addMarker("cache-miss");
                    // Cache miss; send off to the network dispatcher.
                    mNetworkQueue.put(request);
                    continue;
                }

                // If it is completely expired, just send it to the network.
                if (entry.isExpired()) {
                    request.addMarker("cache-hit-expired");
                    request.setCacheEntry(entry);
                    mNetworkQueue.put(request);
                    continue;
                }

                // We have a cache hit; parse its data for delivery back to the request.
                request.addMarker("cache-hit");
                Response<?> response = request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
                request.addMarker("cache-hit-parsed");

                if (!entry.refreshNeeded()) {
                    // Completely unexpired cache hit. Just deliver the response.
                    mDelivery.postResponse(request, response);
                } else {
                    // Soft-expired cache hit. We can deliver the cached response,
                    // but we need to also send the request to the network for
                    // refreshing.
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);

                    // Mark the response as intermediate.
                    response.intermediate = true;

                    // Post the intermediate response back to the user and have
                    // the delivery then forward the request along to the network.
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                                // Not much we can do about this.
                            }
                        }
                    });
                }

            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }
        }
    }
```
- 从`mCacheQueue`中阻塞地获取`Request`，直到该队列中有元素。
- 如果在轮到该`Request`执行时，它已经被`Cancel`了，那么调用`Request#finish`方法，并取队列的下一个`Request`进行操作。
- 尝试从缓存管理类`mCache`中取出该`Request`对应的缓存，如果缓存为空`(entry == null)`，那么将`Request`加入到网络队列；如果缓存过期`(entry.isExpired())`，那么给`Request`设置该缓存，之后加入到网络队列，这两种情况最终都会继续取缓存队列的下一个`Request`进行操作。
- 如果都不是上面的情况，那么通过`Cache.Entry`中的`data`和`header`解析构建`NetworkResponse`，然后回调给`Request`解析，最终得到`Response`
 - 如果缓存需要刷新`(!entry.refreshNeeded())`，那么给`Request`设置`cacheEntry`，在给使用者回调该结果后，还要再把该`Request`加入到网络队列。
 - 如果缓存不需要刷新，那么直接返回即可。

## 4.2 `NetworkDispatcher`
网络线程的处理方式和缓存线程类似，它的构造函数包括：
- `queue`：网络队列
- `network`：网络框架
- `cache`：缓存管理类
- `delivery`：递送类

```
    @Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            Request<?> request;
            try {
                // Take a request from the queue.
                request = mQueue.take();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }

                addTrafficStatsTag(request);

                // Perform the network request.
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }

    private void parseAndDeliverNetworkError(Request<?> request, VolleyError error) {
        error = request.parseNetworkError(error);
        mDelivery.postError(request, error);
    }
```
- 先判断`Request`是否被`Cancel`，如果是，那么直接回调`#finish`，这和上面类似。
- 调用`Network`请求，得到`NetworkResponse`。
- 判断这个`NetworkResponse`是否是`304`并且已经递送给使用者了，如果是，那么直接调用`#finish`，退出。
- 通过`Request#parseNetworkResponse`将`NetworkResponse`解析为`Response`。
- 判断`Request`是否需要缓存，如果需要缓存，那么调用缓存管理类`mCache`保存缓存。
- 通过`mDelivery`递送最终的结果。
- 如果在上述的请求中发生了异常，那么会通过`mDelivery`发送错误给使用者。

# 五、返回结果
从上面的两个`Thread`的处理过程来看，对于从队列中取出的`Request`，最终的处理方式无非有这两种：一种是调用`Request#finish(xxx)`，另一种是通过`mDelivery`。
## 5.1 `Request#finish(xxx)`
```
    //Request.java
   void finish(final String tag) {
        if (mRequestQueue != null) {
            mRequestQueue.finish(this);
        }
    }

    //RequestQueue.java
    <T> void finish(Request<T> request) {
        // Remove from the set of requests currently being processed.
        synchronized (mCurrentRequests) {
            mCurrentRequests.remove(request);
        }
        synchronized (mFinishedListeners) {
          for (RequestFinishedListener<T> listener : mFinishedListeners) {
            listener.onRequestFinished(request);
          }
        }

        if (request.shouldCache()) {
            synchronized (mWaitingRequests) {
                String cacheKey = request.getCacheKey();
                Queue<Request<?>> waitingRequests = mWaitingRequests.remove(cacheKey);
                if (waitingRequests != null) {
                    if (VolleyLog.DEBUG) {
                        VolleyLog.v("Releasing %d waiting requests for cacheKey=%s.",
                                waitingRequests.size(), cacheKey);
                    }
                    // Process all queued up requests. They won't be considered as in flight, but
                    // that's not a problem as the cache has been primed by 'request'.
                    mCacheQueue.addAll(waitingRequests);
                }
            }
        }
    }
```
这里面涉及到两个集合`mCurrentRequests`和`mWaitingRequests`：
- `Set<Request<?>>`：所有被添加的`Request`都进入这一集合，这样在`RequestQueue#cancelAll`时，就是取消这个集合当中的队列。
- `Map<String, Queue<Request<?>>>`：它的`Key`是每个`Request`的`getCacheKey`，具有相同`key`的`Request`会被放在同一个队列当中。

## 5.2 `ResponseDelivery`
```
public interface ResponseDelivery {
    /**
     * Parses a response from the network or cache and delivers it.
     */
    public void postResponse(Request<?> request, Response<?> response);

    /**
     * Parses a response from the network or cache and delivers it. The provided
     * Runnable will be executed after delivery.
     */
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable);

    /**
     * Posts an error for the given request.
     */
    public void postError(Request<?> request, VolleyError error);
}
```
它的默认实现是`ExecutorDelivery`，它负责将请求的结果返回给使用者，它所有的方法，最终都会走到内部的`ResponseDeliveryRunnable`的`run()`方法当中，该`run()`方法运行所在的线程和构建`ExecutorDelivery`所使用的`handler`有关：
```
    private class ResponseDeliveryRunnable implements Runnable {
        private final Request mRequest;
        private final Response mResponse;
        private final Runnable mRunnable;

        public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
            mRequest = request;
            mResponse = response;
            mRunnable = runnable;
        }

        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            // If this request has canceled, finish it and don't deliver.
            if (mRequest.isCanceled()) {
                mRequest.finish("canceled-at-delivery");
                return;
            }

            // Deliver a normal response or error, depending.
            if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }

            // If this is an intermediate response, add a marker, otherwise we're done
            // and the request can be finished.
            if (mResponse.intermediate) {
                mRequest.addMarker("intermediate-response");
            } else {
                mRequest.finish("done");
            }

            // If we have been provided a post-delivery runnable, run it.
            if (mRunnable != null) {
                mRunnable.run();
            }
       }
    }
```

# 六、`RequestQueue`
先看下`Volley`，它其中有一个静态方法`RequestQueue newRequestQueue(Context, HttpStack)`，最终会调用`RequestQueue#start()`。
```
    public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);
        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }
        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }
        Network network = new BasicNetwork(stack);
        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();
        return queue;
    }
```
首先看一下`RequestQueue`的构造函数，它会构造四个`NetworkDispatcher`。
```
    /**
     * Creates the worker pool. Processing will not begin until {@link #start()} is called.
     *
     * @param cache A Cache to use for persisting responses to disk
     * @param network A Network interface for performing HTTP requests
     * @param threadPoolSize Number of network dispatcher threads to create
     * @param delivery A ResponseDelivery interface for posting responses and errors
     */
    public RequestQueue(Cache cache, Network network, int threadPoolSize,
            ResponseDelivery delivery) {
        mCache = cache;
        mNetwork = network;
        mDispatchers = new NetworkDispatcher[threadPoolSize];
        mDelivery = delivery;
    }
```
之后再看一下`start()`方法，它会启动缓存线程，并依次启动网络线程。
```
    /**
     * Starts the dispatchers in this queue.
     */
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
```
当我们调用这个`RequestQueue`的添加方法时：
```
    /**
     * Adds a Request to the dispatch queue.
     * @param request The request to service
     * @return The passed-in request
     */
    public <T> Request<T> add(Request<T> request) {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }

        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }

        // Insert request into stage if there's already a request with the same cache key in flight.
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();
            if (mWaitingRequests.containsKey(cacheKey)) {
                // There is already a request in flight. Queue up.
                Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new LinkedList<Request<?>>();
                }
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
                }
            } else {
                // Insert 'null' queue for this cacheKey, indicating there is now a request in
                // flight.
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }
    }
```
