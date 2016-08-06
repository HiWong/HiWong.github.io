---
layout: post
title: "Retrofit源码解析"
date: 2016-08-06 09:18:14 +0800
comments: true
categories: android_deep_analysis
---
###引言
Retrofit 是 Github 上面 squre 组织开发的一个类型安全的 Http 客户端,它可以在 Java 和 Android 上面使用,并且对Java8做好了适配。Retrofit 将描述
请求的接口转换为对象,然后再由该对象去请求后台。在有了Android Async Http和Volley之后，rxjava+Retrofit+okhttp的网络请求框架能够流行起来呢?一个最主要的原因是它在保证效率的同时，使Http请求变得简单，而且代码的可读性更好<!--more-->。  

###1.rxjava+Retrofit+okhttp使用

以获取考拉分类内容为例，其接口定义如下:  

1)请求方式  
GET http://open.kaolafm.com/v1/content/list?appid={appid}&sign={sign}&openid={openid}&cid={cid}&page={page}&size={size}  
2)认证参数  
参数名称	类型	是否必需	描述  
appid	string	是	应用id  
sign	string	是	签名  
openid	string	是	设备标识  
3)请求参数  
参数名称	 类型	是否必需	描述  
cid	    string	 是	    分类id  
page	string	 否	    页数 默认为1  
size	string	 否	    每页数据条数 默认为10 最大50  

返回结果示例如下:  

```
{
"result": {
"total": "36",
"next": "2",
"prev": "-1",
"list": [
  {
    "cid": "1100000000134",
    "name": "头条新闻",
    "img": "http://image.kaolafm.net/mz/images/201411/1e98d14e-8451-4c96-ad40-b781bd089527/default.jpg",
    "description": "聚焦重大新闻，还原事实真相",
    "type": "0",
    "categoryId": "116",
    "categoryName": "资讯"
  },
  {
    "cid": "1100000000058",
    "name": "V言耸听",
    "img": "http://image.kaolafm.net/mz/images/201406/142eb56f-1c95-4360-a25b-98c13286bb6e/default.jpg",
    "description": "微：微天下；言：语言犀利 言之有物；耸：大V发声 挺拔耸立 观点明晰；听：带您倾听更直观的世界。这里是聚焦时事热点，聆听大V点评的考拉微言耸听",
    "type": "0",
    "categoryId": "116",
    "categoryName": "资讯"
  },
  {
    "cid": "1100000000279",
    "name": "新闻7点整",
    "img": "http://image.kaolafm.net/mz/images/201409/0d4681cc-bded-4692-a2f1-cd06513ae082/default.jpg",
    "description": "这里有最严肃的新闻，这里有最调侃的语言，尽在《新闻7点整》，每天听尽天下事。",
    "type": "0",
    "categoryId": "116",
    "categoryName": "资讯"
  }
]
},
"requestId": "zcrzd94051416464578806827"
}    

```
使用rxjava+Retrofit+okhttp的话，只需要以下三步,第一，声明Client:  

```java
public class KaolaClient {

  protected static volatile KaolaApi api;

  public static KaolaApi get() {
    if (null == api) {
      synchronized (KaolaClient.class) {
        if (null == api) {
          Retrofit retrofit = new Retrofit.Builder().client(StethoHelper.getClient())
              .baseUrl(KaolaConfig.BASE_URL)
              .addConverterFactory(GsonConverterFactory.create())
              .addCallAdapterFactory(RxJavaCallAdapterFactory.createWithScheduler(Schedulers.io()))
              .build();
          api = retrofit.create(KaolaApi.class);
        }
      }
    }
    return api;
  }
}

```
第二，定义号接口：  

```java
//所有跟Kaola有关的接口都可以定义在这里
public interface KaolaApi {

     ...

    @GET("content/list")
    Observable<KaolaSubCategoryItem> getCategoryContent(@Query("appid") String appid,
                                                        @Query("sign") String sign, @Query("openid") String openid, @Query("cid") String cid,
                                                        @Query("page") String page, @Query("size") String size);

   ...
}

```

第三，调用:  

```java
KaolaClient.get().getCategoryContent(appid, sign, MusicDataManager.getInstance().getKaolaOpenid(), categoryId,
                        String.valueOf(page), String.valueOf(DEFAULT_PAGE_SIZE))
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<KaolaSubCategoryItem>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {
                        musicAlbumRecyclerView.setRefreshing(false);
                    }

                    @Override
                    public void onNext(KaolaSubCategoryItem kaolaSubCategoryItem) {
                        musicAlbumAdapter.setData(
                                FMHelper.transformKaolaAlbumBean(kaolaSubCategoryItem.getResult().getList()));
                    }
                });

```

显然，代码的可读性大大加强了。  

###2.准备知识:Java动态代理

代理模式是常用的设计模式，其特征是代理类与委托类具有相同的接口，在具体实现上，有静态代理和动态代理之分。代理类与委托类之间通常会存在关联关系，一个代理类的对象与一个委托类的对象关联，代理类的对象本身并不真正实现服务，而是通过调用委托类的对象的相关方法，来提供特定的服务，也就是说代理类主要负责为委托类预处理消息、过滤消息、把消息转发给委托类，以及事后处理消息等。  

如下是一个Java静态代理的模式:  

{% img /images/android/retrofit/proxy.png %}

显然，如果我们要在所有实现接口的对象执行完该方法之前和之后，都进行同样的操作(如打log),则每增加一个接口，就需要增加一个代理类,显然这样太麻烦。  

如果使用动态代理的话，就可以不用重复定义,还是刚刚那个例子，可以这样处理:  

```java
public class LogHandler implements InvocationHandler{

    private Object targetObject;

    public Object newProxyInstance(Object targetObject){
        this.targetObject=targetObject;
        return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(),
                targetObject.getClass().getInterfaces(),this);
    }

    public Object invoke(Object proxy, Method method, Object[]args) throws Throwable{
        doSomethingBefore();
        Object ret;
        try{
            ret=method.invoke(targetObject,args);
        }catch (Exception ex){
            ex.printStackTrace();
            throw ex;
        }

        doSomethingAfter();

        return ret;

    }


}

```
使用时也很简单：  

```java
public class Client {

    public static void main(String[]args){
        LogHandler logHandler=new LogHandler();
        UserManager userManager=(UserManager)logHandler.newProxyInstance(new UserManagerImpl());
        userManager.addUser("1234","Hillary Clinton");

        DataManager dataManager=(DataMangaer)logHandler.newProxyInstance(new DataManagerImpl());
        dataManager.refreshData("Donald Trump");
    }

}

```
###3.Retrofit原理

前面分析了动态代理，那么回到Http请求这件时上来，所有的Http请求都有一个共同特征:在发送请求之前需要解析参数,在发送请求之后需要处理结果(一般是将json字符串转化为java bean),结合上面的动态代理，我们很自然的想到使用Java动态代理来处理。  

实际上，Retrofit在生成KaolaApi对象的过程中就是这样处理的,下面是Retrofit#create()的代码:  

```java
    public <T> T create(final Class<T> service) {
        Utils.validateServiceInterface(service);
        if (validateEagerly) {
            eagerlyValidateMethods(service);
        }
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
                new InvocationHandler() {
                    private final Platform platform = Platform.get();

                    @Override public Object invoke(Object proxy, Method method, Object... args)
                            throws Throwable {
                        // If the method is a method from Object then defer to normal invocation.
                        if (method.getDeclaringClass() == Object.class) {
                            return method.invoke(this, args);
                        }
                        //只对Java8有效
                        if (platform.isDefaultMethod(method)) {
                            return platform.invokeDefaultMethod(method, service, proxy, args);
                        }
                        ServiceMethod serviceMethod = loadServiceMethod(method);
                        OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
                        return serviceMethod.callAdapter.adapt(okHttpCall);
                    }
                });
    }


```
显然是使用了Java动态代理的方法，到这里可以清晰看到Retrofit的总体过程:在loadServiceMethod中解析http请求参数，然后利用解析出的参数生成Request，在调用okhttp去真正进行请求，最后利用CallAdapter解析结果。  


为了让大家先有一个整体的认识，这里先给出一个核心类的UML图:  

{% img /images/android/retrofit/OkHttpCall.png %}

###4.doSomethingBefore-->parseParameters()

进入loadServiceMethod()方法中，代码如下:  

```java
ServiceMethod loadServiceMethod(Method method) {
        ServiceMethod result;
        synchronized (serviceMethodCache) {
            result = serviceMethodCache.get(method);
            if (result == null) {
                result = new ServiceMethod.Builder(this, method).build();
                serviceMethodCache.put(method, result);
            }
        }
        return result;
    }
```
这里就是如如果在缓存中没有找到，就需要生成一个ServiceMethod对象，先进入Builder()中:  

```java
 public Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }

```
代码很简单，就是获取方法注解和参数类型,以及参数注解。  

build()的代码如下:  

```java

   public ServiceMethod build() {
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }
      responseConverter = createResponseConverter();

      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      if (httpMethod == null) {
        throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
      }

      if (!hasBody) {
        if (isMultipart) {
          throw methodError(
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
        }
      }

      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }

      if (relativeUrl == null && !gotUrl) {
        throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
      }
      if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError("Non-body HTTP method cannot contain @Body.");
      }
      if (isFormEncoded && !gotField) {
        throw methodError("Form-encoded method must contain at least one @Field.");
      }
      if (isMultipart && !gotPart) {
        throw methodError("Multipart method must contain at least one @Part.");
      }

      return new ServiceMethod<>(this);
    }

```
其中parseMethodAnnotation就是解析方法注解的:  

```java
   private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
        if (!Void.class.equals(responseType)) {
          throw methodError("HEAD method must use Void as response type.");
        }
      } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
      } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
      } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
      } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
      } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
      } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
          throw methodError("@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
      } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isMultipart = true;
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }

```
显然，其中主要就是解析HTTP Method和头部参数等。而parameterHandlers[p]=parseParameter(p,parameterType,parameterAnnotations);就是用来解析Http Method和头部参数等之外的接口参数。  

###5.okhttp执行请求

在解析号参数后，就需要生成Request了，那在哪里生成了呢?  
回到Retrofit#create()中来，注意下面这句代码:  

```java
OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
```
我们进入OkHttpCall中，代码如下:  

```java


final class OkHttpCall<T> implements Call<T> {
  private final ServiceMethod<T> serviceMethod;
  private final Object[] args;

  private volatile boolean canceled;

  // All guarded by this.
  private okhttp3.Call rawCall;
  private Throwable creationFailure; // Either a RuntimeException or IOException.
  private boolean executed;

  OkHttpCall(ServiceMethod<T> serviceMethod, Object[] args) {
    this.serviceMethod = serviceMethod;
    this.args = args;
  }

  @SuppressWarnings("CloneDoesntCallSuperClone") // We are a final type & this saves clearing state.
  @Override public OkHttpCall<T> clone() {
    return new OkHttpCall<>(serviceMethod, args);
  }

  @Override public synchronized Request request() {
    okhttp3.Call call = rawCall;
    if (call != null) {
      return call.request();
    }
    if (creationFailure != null) {
      if (creationFailure instanceof IOException) {
        throw new RuntimeException("Unable to create request.", creationFailure);
      } else {
        throw (RuntimeException) creationFailure;
      }
    }
    try {
      return (rawCall = createRawCall()).request();
    } catch (RuntimeException e) {
      creationFailure = e;
      throw e;
    } catch (IOException e) {
      creationFailure = e;
      throw new RuntimeException("Unable to create request.", e);
    }
  }

  @Override public void enqueue(final Callback<T> callback) {
    if (callback == null) throw new NullPointerException("callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }

    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }

  @Override public synchronized boolean isExecuted() {
    return executed;
  }

  @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      if (creationFailure != null) {
        if (creationFailure instanceof IOException) {
          throw (IOException) creationFailure;
        } else {
          throw (RuntimeException) creationFailure;
        }
      }

      call = rawCall;
      if (call == null) {
        try {
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException e) {
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }
    //此处有call.execute()
    return parseResponse(call.execute());
  }

  private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }

  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }

  public void cancel() {
    canceled = true;

    okhttp3.Call call;
    synchronized (this) {
      call = rawCall;
    }
    if (call != null) {
      call.cancel();
    }
  }

  @Override public boolean isCanceled() {
    return canceled;
  }

  static final class NoContentResponseBody extends ResponseBody {
    private final MediaType contentType;
    private final long contentLength;

    NoContentResponseBody(MediaType contentType, long contentLength) {
      this.contentType = contentType;
      this.contentLength = contentLength;
    }

    @Override public MediaType contentType() {
      return contentType;
    }

    @Override public long contentLength() {
      return contentLength;
    }

    @Override public BufferedSource source() {
      throw new IllegalStateException("Cannot read raw response body of a converted body.");
    }
  }

  static final class ExceptionCatchingRequestBody extends ResponseBody {
    private final ResponseBody delegate;
    IOException thrownException;

    ExceptionCatchingRequestBody(ResponseBody delegate) {
      this.delegate = delegate;
    }

    @Override public MediaType contentType() {
      return delegate.contentType();
    }

    @Override public long contentLength() {
      return delegate.contentLength();
    }

    @Override public BufferedSource source() {
      return Okio.buffer(new ForwardingSource(delegate.source()) {
        @Override public long read(Buffer sink, long byteCount) throws IOException {
          try {
            return super.read(sink, byteCount);
          } catch (IOException e) {
            thrownException = e;
            throw e;
          }
        }
      });
    }

    @Override public void close() {
      delegate.close();
    }

    void throwIfCaught() throws IOException {
      if (thrownException != null) {
        throw thrownException;
      }
    }
  }
}


```
显然它完整地实现了Call接口，其中Request request();方法就是用于生成Request的，而execute()方法是用于同步执行Http请求，enqueue(Callback<T>)则是用于异步执行Http请求。  
在接口Call的定义中清楚的注明了各方法的作用:  

```java

public interface Call<T> extends Cloneable {
  /**
   * Synchronously send the request and return its response.
   *
   * @throws IOException if a problem occurred talking to the server.
   * @throws RuntimeException (and subclasses) if an unexpected error occurs creating the request
   * or decoding the response.
   */
  Response<T> execute() throws IOException;

  /**
   * Asynchronously send the request and notify {@code callback} of its response or if an error
   * occurred talking to the server, creating the request, or processing the response.
   */
  void enqueue(Callback<T> callback);

  /**
   * Returns true if this call has been either {@linkplain #execute() executed} or {@linkplain
   * #enqueue(Callback) enqueued}. It is an error to execute or enqueue a call more than once.
   */
  boolean isExecuted();

  /**
   * Cancel this call. An attempt will be made to cancel in-flight calls, and if the call has not
   * yet been executed it never will be.
   */
  void cancel();

  /** True if {@link #cancel()} was called. */
  boolean isCanceled();

  /**
   * Create a new, identical call to this one which can be enqueued or executed even if this call
   * has already been.
   */
  Call<T> clone();

  /** The original HTTP request. */
  Request request();
}

```
再回到OkHttpCall类中，可以明显的看出来它是一个代理类，因为真正执行Http请求的是一个okhttp3.Call对象。那它是如何生成的呢，在request()方法中有return (rawCall = createRawCall()).request();这句，进入到createRawCall()中,代码如下:  

```java

private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }

```
可以看到在serviceMethod.toRequest()中生成了Request对象.其代码如下:  

```java
 /** Builds an HTTP request from method arguments. */
  Request toRequest(Object... args) throws IOException {
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);

    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args != null ? args.length : 0;
    if (argumentCount != handlers.length) {
      throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }

    for (int p = 0; p < argumentCount; p++) {
      handlers[p].apply(requestBuilder, args[p]);
    }

    return requestBuilder.build();
  }

```
显然就是将刚刚解析出的httpMethod,baseUrl,relativeUrl,headers,contentType,hasBody,isFormEncodes,parameterHandlers等参数构造出一个Request对象.  

回到createRawCall()方法中，那serviceMethod中的callFactory是什么呢?  
在ServiceMethod#Builder()中可以看到是来自Retrofit中，我们看一下Retrofit#build()代码:  

```java
 public Retrofit build() {
            if (baseUrl == null) {
                throw new IllegalStateException("Base URL required.");
            }

            okhttp3.Call.Factory callFactory = this.callFactory;
            if (callFactory == null) {
                callFactory = new OkHttpClient();
            }

            Executor callbackExecutor = this.callbackExecutor;
            if (callbackExecutor == null) {
                callbackExecutor = platform.defaultCallbackExecutor();
            }

            // Make a defensive copy of the adapters and add the default Call adapter.
            List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
            adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

            // Make a defensive copy of the converters.
            List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

            return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
                    callbackExecutor, validateEagerly);
        }

```
可以看到，除非用户自己指定了callFactory,否则默认的是一个OkHttpClient对象。  

那OkHttpCall对象何时会被调用呢?  
回到Retrofit中，剩下的代码只有return serviceMethod.callAdapter.adapt(okHttpCall);所以肯定在这里面调用到了。  

先看一下CallAdapter的定义:  

```java
public interface CallAdapter<T> {
  /**
   * Returns the value type that this adapter uses when converting the HTTP response body to a Java
   * object. For example, the response type for {@code Call<Repo>} is {@code Repo}. This type
   * is used to prepare the {@code call} passed to {@code #adapt}.
   * <p>
   * Note: This is typically not the same type as the {@code returnType} provided to this call
   * adapter's factory.
   */
  Type responseType();

  /**
   * Returns an instance of {@code T} which delegates to {@code call}.
   * <p>
   * For example, given an instance for a hypothetical utility, {@code Async}, this instance would
   * return a new {@code Async<R>} which invoked {@code call} when run.
   * <pre>{@code
   * &#64;Override
   * public <R> Async<R> adapt(final Call<R> call) {
   *   return Async.create(new Callable<Response<R>>() {
   *     &#64;Override
   *     public Response<R> call() throws Exception {
   *       return call.execute();
   *     }
   *   });
   * }
   * }</pre>
   */
  <R> T adapt(Call<R> call);

  /**
   * Creates {@link CallAdapter} instances based on the return type of {@linkplain
   * Retrofit#create(Class) the service interface} methods.
   */
  abstract class Factory {
    /**
     * Returns a call adapter for interface methods that return {@code returnType}, or null if it
     * cannot be handled by this factory.
     */
    public abstract CallAdapter<?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);

    /**
     * Extract the upper bound of the generic parameter at {@code index} from {@code type}. For
     * example, index 1 of {@code Map<String, ? extends Runnable>} returns {@code Runnable}.
     */
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    /**
     * Extract the raw class type from {@code type}. For example, the type representing
     * {@code List<? extends Runnable>} returns {@code List.class}.
     */
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}

```
CallAdapter本身的方法很简单，就是一个返回结果类型方法和适配方法(所谓适配，就是将A类型转换为B类型),重点是里面的工厂类，后面有多个类是实现CallAdapter.Factory的.  

serviceMethod中的callAdapter来自Retrofit,那Retrofit中的callAdapter是如何来的呢?  

注意到Retrofit.Builder#addCallAdapterFactory()方法，代码如下:  

```java
  public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
            adapterFactories.add(checkNotNull(factory, "factory == null"));
            return this;
        }

```
显然用户可指定，那如果用户没有指定，有默认的吗?  
注意到Retrofit.Builder#build()中以下代码:  

```java
  // Make a defensive copy of the adapters and add the default Call adapter.
  List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
  adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

```
显然，如果没有指定的话，有defaultCallAdapterFactory,进入该方法:  

```java

CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
    if (callbackExecutor != null) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    return DefaultCallAdapterFactory.INSTANCE;
  }

```
显然，如果用户指定了callbackExecutor,则使用ExecutorCallAdapterFactory,否则使用DefaultCallAdapterFactory.INSTANCE;  

先看ExecutorCallAdapterFactory的代码:  

```java
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public CallAdapter<Call<?>> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public <R> Call<R> adapt(Call<R> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }

  static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }

    @Override public void enqueue(final Callback<T> callback) {
      if (callback == null) throw new NullPointerException("callback == null");

      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(final Call<T> call, final Response<T> response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(call, new IOException("Canceled"));
              } else {
                callback.onResponse(call, response);
              }
            }
          });
        }

        @Override public void onFailure(final Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(call, t);
            }
          });
        }
      });
    }

    @Override public boolean isExecuted() {
      return delegate.isExecuted();
    }

    @Override public Response<T> execute() throws IOException {
      return delegate.execute();
    }

    @Override public void cancel() {
      delegate.cancel();
    }

    @Override public boolean isCanceled() {
      return delegate.isCanceled();
    }

    @SuppressWarnings("CloneDoesntCallSuperClone") // Performing deep clone.
    @Override public Call<T> clone() {
      return new ExecutorCallbackCall<>(callbackExecutor, delegate.clone());
    }

    @Override public Request request() {
      return delegate.request();
    }
  }
}

```
显然，OkHttpCall对象在其中作为delegate存在，OkHttpCall对象的调用也是在其中完成的。  

再看以下DefaultCallAdapterFactory的代码：  

```java
final class DefaultCallAdapterFactory extends CallAdapter.Factory {
  static final CallAdapter.Factory INSTANCE = new DefaultCallAdapterFactory();

  @Override
  public CallAdapter<?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }

    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public <R> Call<R> adapt(Call<R> call) {
        return call;
      }
    };
  }
}

```
显然，除了对返回类型进行了处理之外，call是原样返回的。  
现在一般配合rxjava使用，注意到KaolaClient中的以下代码:  

```java

.addCallAdapterFactory(RxJavaCallAdapterFactory.createWithScheduler(Schedulers.io()))

```
下面我们看一下RxJavaCallAdapterFactory的代码:  

```java

public final class RxJavaCallAdapterFactory extends CallAdapter.Factory {
  /**
   * Returns an instance which creates synchronous observables that do not operate on any scheduler
   * by default.
   */
  public static RxJavaCallAdapterFactory create() {
    return new RxJavaCallAdapterFactory(null);
  }

  /**
   * Returns an instance which creates synchronous observables that
   * {@linkplain Observable#subscribeOn(Scheduler) subscribe on} {@code scheduler} by default.
   */
  public static RxJavaCallAdapterFactory createWithScheduler(Scheduler scheduler) {
    if (scheduler == null) throw new NullPointerException("scheduler == null");
    return new RxJavaCallAdapterFactory(scheduler);
  }

  private final Scheduler scheduler;

  private RxJavaCallAdapterFactory(Scheduler scheduler) {
    this.scheduler = scheduler;
  }

  @Override
  public CallAdapter<?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    Class<?> rawType = getRawType(returnType);
    String canonicalName = rawType.getCanonicalName();
    boolean isSingle = "rx.Single".equals(canonicalName);
    boolean isCompletable = "rx.Completable".equals(canonicalName);
    if (rawType != Observable.class && !isSingle && !isCompletable) {
      return null;
    }
    if (!isCompletable && !(returnType instanceof ParameterizedType)) {
      String name = isSingle ? "Single" : "Observable";
      throw new IllegalStateException(name + " return type must be parameterized"
          + " as " + name + "<Foo> or " + name + "<? extends Foo>");
    }

    if (isCompletable) {
      // Add Completable-converter wrapper from a separate class. This defers classloading such that
      // regular Observable operation can be leveraged without relying on this unstable RxJava API.
      // Note that this has to be done separately since Completable doesn't have a parametrized
      // type.
      return CompletableHelper.createCallAdapter(scheduler);
    }

    CallAdapter<Observable<?>> callAdapter = getCallAdapter(returnType, scheduler);
    if (isSingle) {
      // Add Single-converter wrapper from a separate class. This defers classloading such that
      // regular Observable operation can be leveraged without relying on this unstable RxJava API.
      return SingleHelper.makeSingle(callAdapter);
    }
    return callAdapter;
  }

  private CallAdapter<Observable<?>> getCallAdapter(Type returnType, Scheduler scheduler) {
    Type observableType = getParameterUpperBound(0, (ParameterizedType) returnType);
    Class<?> rawObservableType = getRawType(observableType);
    if (rawObservableType == Response.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException("Response must be parameterized"
            + " as Response<Foo> or Response<? extends Foo>");
      }
      Type responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
      return new ResponseCallAdapter(responseType, scheduler);
    }

    if (rawObservableType == Result.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException("Result must be parameterized"
            + " as Result<Foo> or Result<? extends Foo>");
      }
      Type responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
      return new ResultCallAdapter(responseType, scheduler);
    }

    return new SimpleCallAdapter(observableType, scheduler);
  }

  static final class CallOnSubscribe<T> implements Observable.OnSubscribe<Response<T>> {
    private final Call<T> originalCall;

    CallOnSubscribe(Call<T> originalCall) {
      this.originalCall = originalCall;
    }

    @Override public void call(final Subscriber<? super Response<T>> subscriber) {
      // Since Call is a one-shot type, clone it for each new subscriber.
      final Call<T> call = originalCall.clone();

      // Attempt to cancel the call if it is still in-flight on unsubscription.
      subscriber.add(Subscriptions.create(new Action0() {
        @Override public void call() {
          call.cancel();
        }
      }));

      try {
        Response<T> response = call.execute();
        if (!subscriber.isUnsubscribed()) {
          subscriber.onNext(response);
        }
      } catch (Throwable t) {
        Exceptions.throwIfFatal(t);
        if (!subscriber.isUnsubscribed()) {
          subscriber.onError(t);
        }
        return;
      }

      if (!subscriber.isUnsubscribed()) {
        subscriber.onCompleted();
      }
    }
  }

  static final class ResponseCallAdapter implements CallAdapter<Observable<?>> {
    private final Type responseType;
    private final Scheduler scheduler;

    ResponseCallAdapter(Type responseType, Scheduler scheduler) {
      this.responseType = responseType;
      this.scheduler = scheduler;
    }

    @Override public Type responseType() {
      return responseType;
    }

    @Override public <R> Observable<Response<R>> adapt(Call<R> call) {
      Observable<Response<R>> observable = Observable.create(new CallOnSubscribe<>(call));
      if (scheduler != null) {
        return observable.subscribeOn(scheduler);
      }
      return observable;
    }
  }

  static final class SimpleCallAdapter implements CallAdapter<Observable<?>> {

    private final Type responseType;
    private final Scheduler scheduler;

    SimpleCallAdapter(Type responseType, Scheduler scheduler) {
      this.responseType = responseType;
      this.scheduler = scheduler;
    }

    @Override public Type responseType() {
      return responseType;
    }

    @Override public <R> Observable<R> adapt(Call<R> call) {
      Observable<R> observable = Observable.create(new CallOnSubscribe<>(call)) //
          .flatMap(new Func1<Response<R>, Observable<R>>() {
            @Override public Observable<R> call(Response<R> response) {
              if (response.isSuccessful()) {
                return Observable.just(response.body());
              }
              return Observable.error(new HttpException(response));
            }
          });
      if (scheduler != null) {
        return observable.subscribeOn(scheduler);
      }
      return observable;
    }
  }

  static final class ResultCallAdapter implements CallAdapter<Observable<?>> {
    private final Type responseType;
    private final Scheduler scheduler;

    ResultCallAdapter(Type responseType, Scheduler scheduler) {
      this.responseType = responseType;
      this.scheduler = scheduler;
    }

    @Override public Type responseType() {
      return responseType;
    }

    @Override public <R> Observable<Result<R>> adapt(Call<R> call) {
      Observable<Result<R>> observable = Observable.create(new CallOnSubscribe<>(call)) //
          .map(new Func1<Response<R>, Result<R>>() {
            @Override public Result<R> call(Response<R> response) {
              return Result.response(response);
            }
          }).onErrorReturn(new Func1<Throwable, Result<R>>() {
            @Override public Result<R> call(Throwable throwable) {
              return Result.error(throwable);
            }
          });
      if (scheduler != null) {
        return observable.subscribeOn(scheduler);
      }
      return observable;
    }
  }
}

```
注意到CallOnSubscribe中的call()方法，里面的 Response<T> response = call.execute(); 就是调用OkHttpCall对象的同步执行方法。  

对比以下，为什么ExecutorCallAdapterFactory()中既有execute()也有enqueue()方法，而RxJavaCallAdapterFactory中却只调用了OkHttpCall的execute()方法呢?  
原因是rxjava调用execute()在工作线程执行后，有一套自己的机制将结果交到观察者线程。  

###6.doSomethingAfter()-->Callback.OnResponse(),Observable.subscribe()

后处理是利用Converter的实现类来完成的，先看一下Converter的定义：  

```java
public interface Converter<F, T> {
  T convert(F value) throws IOException;

  /** Creates {@link Converter} instances based on a type and target usage. */
  abstract class Factory {
    /**
     * Returns a {@link Converter} for converting an HTTP response body to {@code type}, or null if
     * {@code type} cannot be handled by this factory. This is used to create converters for
     * response types such as {@code SimpleResponse} from a {@code Call<SimpleResponse>}
     * declaration.
     */
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      return null;
    }

    /**
     * Returns a {@link Converter} for converting {@code type} to an HTTP request body, or null if
     * {@code type} cannot be handled by this factory. This is used to create converters for types
     * specified by {@link Body @Body}, {@link Part @Part}, and {@link PartMap @PartMap}
     * values.
     */
    public Converter<?, RequestBody> requestBodyConverter(Type type,
        Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
      return null;
    }

    /**
     * Returns a {@link Converter} for converting {@code type} to a {@link String}, or null if
     * {@code type} cannot be handled by this factory. This is used to create converters for types
     * specified by {@link Field @Field}, {@link FieldMap @FieldMap} values,
     * {@link Header @Header}, {@link Path @Path}, {@link Query @Query}, and
     * {@link QueryMap @QueryMap} values.
     */
    public Converter<?, String> stringConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      return null;
    }
  }
}

```
接口非常简单，就是一个转换方法和一个工厂类。  
实现Converter.Factory的有GsonConverterFactory和JacksonConverterFactory,这里只分析GsonConverterFactory:  

```java
public final class GsonConverterFactory extends Converter.Factory {
  /**
   * Create an instance using a default {@link Gson} instance for conversion. Encoding to JSON and
   * decoding from JSON (when no charset is specified by a header) will use UTF-8.
   */
  public static GsonConverterFactory create() {
    return create(new Gson());
  }

  /**
   * Create an instance using {@code gson} for conversion. Encoding to JSON and
   * decoding from JSON (when no charset is specified by a header) will use UTF-8.
   */
  public static GsonConverterFactory create(Gson gson) {
    return new GsonConverterFactory(gson);
  }

  private final Gson gson;

  private GsonConverterFactory(Gson gson) {
    if (gson == null) throw new NullPointerException("gson == null");
    this.gson = gson;
  }

  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);
  }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonRequestBodyConverter<>(gson, adapter);
  }
}

```
主体就是利用gson将Response数据转换为指定的类型，不再赘述。  



