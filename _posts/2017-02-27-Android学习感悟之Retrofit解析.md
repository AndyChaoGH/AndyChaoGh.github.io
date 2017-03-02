---
layout: post
title: Android感悟之Retrofit解析
author: arvinljw
---

本篇包含Retrofit源码解析以及我自己简单封装后的用法

# 简介

Retrofit+RxJava在网络请求上的使用，已经火了快一年了。最开始的时候也有去简单的了解如何使用和看过源码分析，stay大神写的[Retrofit分析-漂亮的解耦套路](http://www.jianshu.com/p/45cb536be2f4)这篇文章让我受益匪浅，现在自己最近也在总结这些东西，而听别人说的，我总觉得少了点什么，总得自己去发现的才是自己的，所以就准备写出这篇文章。

首先这篇文章是在使用过Retrofit的基础上，才能继续阅读的，所以基础的为什么要这么写就不再一一叙述了。

### 1、首先需要创建一个Retrofit对象：

```
Retrofit retrofit = new Retrofit.Builder()
        .client(okHttpClient)
        .baseUrl(getBaseUrl())
        .addConverterFactory(converterFactory)
        .addCallAdapterFactory(rxJavaCallAdapterFactory)
        .build();
```

其中通过Retrofit的Builder的build()方法来创建一个Retrofit实例，中间还设置了一些参数，阅读源码后可知道（源码都比较简单，都是简单的判断和赋值）：

* client：默认也是一个OkHttpClient，而我们自己创建的一个OkHttpClient可以通过拦截器统一动态添加header；
* baseUrl：这里的url会先经过验证是否是正确的url，且需要以"/"结尾才行；
* ConverterFactory：这里传入的是我们自定义的convert，这个是用于转换http请求的request和response中参数的工具。这里可以根据项目中服务器的要求，请求使用Gson数据即可，返回后再自行解析为想要的类型。
* CallAdapterFactory：回调为相应的数据类型，这里使用的是RxJava的CallAdapterFactory，则返回Observer<T>类型的数据；默认Android中是返回Call<T>类型的数据;
* build()：该方法就是保证可不是必须配置的参数，射上一个默认值，并返回一个Retrofit对象。

### 2、造出一个接口Api的动态代理：

```
private T api;
private Class<T> clazz;

clazz = (Class<T>) ((ParameterizedType) getClass()
        .getGenericSuperclass()).getActualTypeArguments()[0];

api = retrofit.create(clazz);
```

这里我使用了泛型，然后create中传入的就是T.class；下面直接上create的代码：

```
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

可以看到简单的验证了一下，返回的是一个传入Class的动态代理，

可以看到，当使用api，调用其我们写的的方法时，都会回调到这个动态代理中，且主要执行的就是最后三句话，我们依次看：

**ServiceMethod serviceMethod = loadServiceMethod(method);**进入源码可以看到

```
private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();

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

这里边使用了一个LinkedHashMap来缓存这个方法，key是被调用的Medthod，value是这个Medthod对应的ServiceMedthod，接着来看看这个ServiceMethod是什么。

```
private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();

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

public ServiceMethod.Builder(Retrofit retrofit, Method method) {
  this.retrofit = retrofit;
  this.method = method;
  this.methodAnnotations = method.getAnnotations();
  this.parameterTypes = method.getGenericParameterTypes();
  this.parameterAnnotationsArray = method.getParameterAnnotations();
}

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

  //参数注解的解析...

  return new ServiceMethod<>(this);
}
```

可以看到这里比较重要的是在ServiceMethod中创建了callAdapter和responseConverter，以及方法上注解的转化（如何转化先不管）；先看看callAdapter如何创建的；

```
private CallAdapter<?> createCallAdapter() {
  Type returnType = method.getGenericReturnType();
  if (Utils.hasUnresolvableType(returnType)) {
    throw methodError(
        "Method return type must not include a type variable or wildcard: %s", returnType);
  }
  if (returnType == void.class) {
    throw methodError("Service methods cannot return void.");
  }
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create call adapter for %s", returnType);
  }
}

public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
  return nextCallAdapter(null, returnType, annotations);
}

public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
    Annotation[] annotations) {
  checkNotNull(returnType, "returnType == null");
  checkNotNull(annotations, "annotations == null");

  int start = adapterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = adapterFactories.size(); i < count; i++) {
    CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
    if (adapter != null) {
      return adapter;
    }
  }

  //构造StringBuilder builder...

  throw new IllegalArgumentException(builder.toString());
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
```

可以看到是根据Api中方法的返回类型来和Retrofit中设置的CallAdapterFactory来比较，如果匹配到了就返回该adapter作为ServiceMethod的callAdapter，没有则抛异常；其中RxJavaCallAdapterFactory匹配就是要保证返回类型必须是Observable，且返回的adapter有三种ResponseCallAdapter、ResultCallAdapter和SimpleCallAdapter，具体的使用会在之后的adapt中看到；

再来看responseConverter是如何创建的：

```
private Converter<ResponseBody, T> createResponseConverter() {
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.responseBodyConverter(responseType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create converter for %s", responseType);
  }
}

public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
  return nextResponseBodyConverter(null, type, annotations);
}

public <T> Converter<ResponseBody, T> nextResponseBodyConverter(Converter.Factory skipPast,
    Type type, Annotation[] annotations) {
  checkNotNull(type, "type == null");
  checkNotNull(annotations, "annotations == null");

  int start = converterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = converterFactories.size(); i < count; i++) {
    Converter<ResponseBody, ?> converter =
        converterFactories.get(i).responseBodyConverter(type, annotations, this);
    if (converter != null) {
      //noinspection unchecked
      return (Converter<ResponseBody, T>) converter;
    }
  }

  //构造StringBuilder builder...

  throw new IllegalArgumentException(builder.toString());
}
```

这个比较简单，直接就是在Retrofit中设置的CoverterFactory中去调用responseBodyConverter方法，只有获取到的ResponseConverter不为null即可，否则会抛异常；

这样loadServiceMethod的过程就是这样，是去创建了缓存了callAdapter和ResponseConverter以及最重要的缓存了Method对应的注解解析；

**OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);**接着看这句话，它创建了一个OkHttpCall，看看做了哪些操作：

```
OkHttpCall(ServiceMethod<T> serviceMethod, Object[] args) {
  this.serviceMethod = serviceMethod;
  this.args = args;
}
```

就是赋值，需要知道的事OkHttpCall继承于Call，用于执行最后的网络请求，好那接着看**serviceMethod.callAdapter.adapt(okHttpCall)**这句，这里的callAdapter以RxJavaCallAdapterFactory为例：调用其adapter方法，传入okHttpCall；就以ResponseCallAdapter为例，其他的也是一样都是将okHttpCall，封装为CallOnSubscribe赋值给Observable的onSubscribe属性；

```
public <R> Observable<Response<R>> adapt(Call<R> call) {
    Observable<Response<R>> observable = Observable.create(new CallOnSubscribe<>(call));
    if (scheduler != null) {
      return observable.subscribeOn(scheduler);
    }
    return observable;
  }
}

static final class CallOnSubscribe<T> implements Observable.OnSubscribe<Response<T>> {
  private final Call<T> originalCall;

  CallOnSubscribe(Call<T> originalCall) {
    this.originalCall = originalCall;
  }

  @Override public void call(final Subscriber<? super Response<T>> subscriber) {
    // Since Call is a one-shot type, clone it for each new subscriber.
    Call<T> call = originalCall.clone();

    // Wrap the call in a helper which handles both unsubscription and backpressure.
    RequestArbiter<T> requestArbiter = new RequestArbiter<>(call, subscriber);
    subscriber.add(requestArbiter);
    subscriber.setProducer(requestArbiter);
  }
}

public static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(RxJavaHooks.onCreate(f));
}

public static <T> Observable.OnSubscribe<T> onCreate(Observable.OnSubscribe<T> onSubscribe) {
    Func1<Observable.OnSubscribe, Observable.OnSubscribe> f = onObservableCreate;
    if (f != null) {
        return f.call(onSubscribe);
    }
    return onSubscribe;
}

protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;
}
```

这里发现只是通过callAdapter返回为一个Observable<T>对象，了解RxJava的知道，还需要subscribe后才会真正的去执行这个方法；例如以登录为例：

```
NetService.getInstance().login(name, password)
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(new Subscriber<UserEntity>() {
    @Override
    public void onCompleted() {
    }

    @Override
    public void onError(Throwable e) {
    }

    @Override
    public void onNext(UserEntity userEntity) {
    }
});
```

这里可以看到有个线程切换的过程，网络请求执行在自线程Subscriber回调在主线程；下面看subscribe中如何实现；

```
public final Subscription subscribe(Subscriber<? super T> subscriber) {
    return Observable.subscribe(subscriber, this);
}

static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
    if (subscriber == null) {
        throw new IllegalArgumentException("subscriber can not be null");
    }
    if (observable.onSubscribe == null) {
        throw new IllegalStateException("onSubscribe function can not be null.");
    }

    // new Subscriber so onStart it
    subscriber.onStart();

    // if not already wrapped
    if (!(subscriber instanceof SafeSubscriber)) {
        // assign to `observer` so we return the protected version
        subscriber = new SafeSubscriber<T>(subscriber);
    }

    try {
        // allow the hook to intercept and/or decorate
        RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber);
        return RxJavaHooks.onObservableReturn(subscriber);
    } catch (Throwable e) {
        //deal Exception...
        return Subscriptions.unsubscribed();
    }
}

public static <T> Observable.OnSubscribe<T> onObservableStart(Observable<T> instance, Observable.OnSubscribe<T> onSubscribe) {
    Func2<Observable, Observable.OnSubscribe, Observable.OnSubscribe> f = onObservableStart;
    if (f != null) {
        return f.call(instance, onSubscribe);
    }
    return onSubscribe;
}

public static Subscription onObservableReturn(Subscription subscription) {
    Func1<Subscription, Subscription> f = onObservableReturn;
    if (f != null) {
        return f.call(subscription);
    }
    return subscription;
}
```
跟着这条线索走下去，发现在subscribe中里边的onSubscibe就是之前传入的用okHttpCall封装的CallOnSubscribe对象，调用它的call将RequestArbiter设置给subscriber。其中RequestArbiter的request会最终调用okHttpCall的execute方法，然后再通过subscriber之行onNext方法，回调到主线程。

```
static final class RequestArbiter<T> extends AtomicBoolean implements Subscription, Producer {
  private final Call<T> call;
  private final Subscriber<? super Response<T>> subscriber;

  RequestArbiter(Call<T> call, Subscriber<? super Response<T>> subscriber) {
    this.call = call;
    this.subscriber = subscriber;
  }

  @Override 
  public void request(long n) {
    if (n < 0) throw new IllegalArgumentException("n < 0: " + n);
    if (n == 0) return; // Nothing to do when requesting 0.
    if (!compareAndSet(false, true)) return; // Request was already triggered.

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

@Override 
public Response<T> execute() throws IOException {
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

  return parseResponse(call.execute());
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

T toResponse(ResponseBody body) throws IOException {
  return responseConverter.convert(body);
}
```

这样就返回了我们需要的数据类型，整个流程就完了。其中比较绕的就是最后的回调过程，目前我也不能很好的解释这一块，但是确实是通过这里执行完结束的。这里边还有一点没有找到的就是RequestConverter的转化，而我们在ServiceMethod中转化解析注解的时候没去关注它，而requestConverter的赋值就正是在那里；而具体的调用是在okHttpCall中的createRawCall()里边的调用toRequest，再使用ParameterHandler.apply方法，对不同的参数，都使用在注解解析那里赋值的converter来转化，这样整体的流程就真正完成了。

下面引用一张Stay大神整理的流程图，梳理一下整个调用过程。

![](../images/retrofit_work_flow.png)

至此整个Retrofit的源码分析到此结束，虽不尽完美，却依旧有所收获。

## Retrofit封装及使用

经过上面源码的分析，主要是清楚了其中的一些细节以及整体流程，下面就来看看具体的使用流程：

1、引入依赖

```
compile 'com.squareup.retrofit2:retrofit:2.2.0'
compile 'com.squareup.retrofit2:converter-gson:2.2.0'
compile 'com.squareup.retrofit2:adapter-rxjava:2.2.0'
compile 'io.reactivex:rxjava:1.2.1'
compile 'io.reactivex:rxandroid:1.2.1'
compile 'com.google.code.gson:gson:2.7'
compile 'com.github.bumptech.glide:okhttp3-integration:1.4.0@aar'
```

这里rxjava继续使用1.x的版本，2.x的还没有去研究，因为目前并没有出现问题（据说会导致backpressure），再来看看封装过后的BaseNet：

```
import android.content.Context;
import android.text.TextUtils;

import com.bumptech.glide.Glide;
import com.bumptech.glide.integration.okhttp3.OkHttpUrlLoader;
import com.bumptech.glide.load.model.GlideUrl;
import com.orhanobut.logger.Logger;

import net.arvin.afbaselibrary.nets.converters.GsonConverterFactory;
import net.arvin.afbaselibrary.utils.CertificateUtil;

import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.ParameterizedType;

import okhttp3.Interceptor;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
import retrofit2.CallAdapter;
import retrofit2.Converter;
import retrofit2.Retrofit;
import retrofit2.adapter.rxjava.RxJavaCallAdapterFactory;

/**
 * created by arvin on 16/11/21 23:06
 * email：1035407623@qq.com
 */
public abstract class BaseNet<T> {
    private T api;
    private Class<T> clazz;
    private OkHttpClient okHttpClient;
    private String token;
    private String deviceId;

    protected Converter.Factory converterFactory;
    protected CallAdapter.Factory rxJavaCallAdapterFactory;

    @SuppressWarnings("unchecked")
    protected BaseNet() {
        converterFactory = GsonConverterFactory.create();
        rxJavaCallAdapterFactory = RxJavaCallAdapterFactory.create();
        clazz = (Class<T>) ((ParameterizedType) getClass()
                .getGenericSuperclass()).getActualTypeArguments()[0];

    }

    public T getApi() {
        if (api == null) {
            initHttpClient();
            Retrofit retrofit = new Retrofit.Builder()
                    .client(okHttpClient)
                    .baseUrl(getBaseUrl())
                    .addConverterFactory(converterFactory)
                    .addCallAdapterFactory(rxJavaCallAdapterFactory)
                    .build();

            api = retrofit.create(clazz);
        }
        return api;
    }

    @SuppressWarnings("ConstantConditions")
    private void initHttpClient() {
        if (okHttpClient == null) {
            OkHttpClient.Builder builder = new OkHttpClient.Builder().addInterceptor(new Interceptor() {
                @Override
                public Response intercept(Chain chain) throws IOException {
                    Request request;
                    if (!TextUtils.isEmpty(token)) {
                        request = chain.request().newBuilder()
                                .addHeader("token", token)
                                .addHeader("version", "1.0")
                                .build();
                    } else {
                        request = chain.request().newBuilder()
                                .addHeader("version", "1.0")
                                .build();
                    }
                    request = dealRequest(request);
                    Logger.i("request" + request);

                    Response response = chain.proceed(request);
                    dealResponse(response);
                    return response;
                }
            });
            if (isNeedHttps()) {
                try {
                    if (getContext() == null) {
                        throw new IllegalArgumentException("Uses custom https' api have to override getContext() Method");
                    }
                    builder.sslSocketFactory(CertificateUtil.setCertificates(getContext(), getCertificateNames()))
                            .hostnameVerifier(org.apache.http.conn.ssl.SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            okHttpClient = builder.build();
        }
    }

    /**
     * 若需要使用https请求,请设置证书信息 getCertificateNames()和getContext()
     */
    protected boolean isNeedHttps() {
        return false;
    }

    protected String[] getCertificateNames() {
        return null;
    }

    protected Context getContext() {
        return null;
    }

    public void setToken(String token) {
        this.token = token;
        clear();
    }

    public String getToken() {
        return token;
    }

    public void setDeviceId(String deviceId) {
        this.deviceId = deviceId;
    }

    public String getDeviceId() {
        return deviceId;
    }

    private void clear() {
        okHttpClient = null;
        api = null;
    }

    protected Request dealRequest(Request request) {
        return request;
    }

    protected void dealResponse(Response response) {
    }

    /**
     * 让Glide支持https
     */
    public void registerHttps() {
        Glide.get(getContext()).register(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory(okHttpClient));
    }

    protected abstract String getBaseUrl();

}
```

其中使用到了泛型，T就是我们的Api接口，clazz为T.class，okhttpClient是我们自己添加拦截器的client，用于动态添加header；还有converter和callAdapter，分别是数据转换器和数据回调方式；若要使用自定义证书的https请求，则可以重写isNeedHttps、getCertificateNames、getContext方法即可；注意证书的位置需要放在assets目录下，因为配置的源码是从assets下去找的，当然也可以修改；CertificateUtil的源码如下：

```
import android.content.Context;
import android.content.res.AssetManager;

import com.orhanobut.logger.Logger;

import java.io.IOException;
import java.io.InputStream;
import java.security.KeyStore;
import java.security.SecureRandom;
import java.security.cert.CertificateFactory;

import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManagerFactory;

/**
 * created by arvin on 16/8/3 10:11
 * email：1035407623@qq.com
 */
public class CertificateUtil {

    private static InputStream[] getCertificatesByAssert(Context context, String... certificateNames) {
        if (context == null) {
            Logger.d("context is empty");
            return null;
        }
        if (certificateNames == null) {
            Logger.d("certificate is empty");
            return null;
        }

        AssetManager assets = context.getAssets();
        InputStream[] certificates = new InputStream[certificateNames.length];
        for (int i = 0; i < certificateNames.length; i++) {
            String certificateName = certificateNames[i];
            try {
                certificates[i] = assets.open(certificateName);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return certificates;
    }

    public static SSLSocketFactory setCertificates(Context context, String... certificateNames) {
        InputStream[] certificates = getCertificatesByAssert(context, certificateNames);
        if (certificates == null) {
            return null;
        }
        try {
            CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
            KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
            keyStore.load(null);
            int index = 0;
            for (InputStream certificate : certificates) {
                String certificateAlias = Integer.toString(index++);
                keyStore.setCertificateEntry(certificateAlias, certificateFactory.generateCertificate(certificate));

                try {
                    if (certificate != null)
                        certificate.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            SSLContext sslContext = SSLContext.getInstance("TLS");

            TrustManagerFactory trustManagerFactory =
                    TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());

            trustManagerFactory.init(keyStore);
            sslContext.init(null, trustManagerFactory.getTrustManagers(), new SecureRandom());
            return sslContext.getSocketFactory();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

其他https相关知识就不作介绍了。最后还有个比较重要的就是回调到主线程的callback之后的处理：

```
import android.content.Context;
import android.content.Intent;
import android.net.ParseException;
import android.widget.Toast;

import com.google.gson.JsonParseException;
import com.orhanobut.logger.Logger;

import net.arvin.afbaselibrary.nets.exceptions.ApiException;
import net.arvin.afbaselibrary.nets.exceptions.ResultException;
import net.arvin.afbaselibrary.utils.ActivityUtil;

import org.json.JSONException;

import java.net.ConnectException;

import retrofit2.adapter.rxjava.HttpException;
import rx.Subscriber;

/**
 * created by arvin on 16/10/24 17:20
 * email：1035407623@qq.com
 */
public abstract class AbsAPICallback<T> extends Subscriber<T> {

    //出错提示
    public final String networkMsg = "服务器开小差";
    public final String parseMsg = "数据解析出错";
    public final String net_connection = "网络连接错误";
    public final String unknownMsg = "未知错误";

    protected AbsAPICallback() {
    }


    @Override
    public void onError(Throwable e) {
        e = getThrowable(e);

        if (e instanceof HttpException) {//HTTP错误

            error(e, ((HttpException) e).code(), networkMsg);

        } else if (e instanceof ResultException) {//服务器返回的错误

            ResultException resultException = (ResultException) e;
            error(e, resultException.getErrCode(), resultException.getMessage());
            resultError(resultException);//处理错误

        } else if (e instanceof JsonParseException || e instanceof JSONException || e instanceof ParseException) {//解析错误

            error(e, ApiException.PARSE_ERROR, parseMsg);

        } else if (e instanceof ConnectException) {
            error(e, ApiException.PARSE_ERROR, net_connection);
        } else {//未知错误
            error(e, ApiException.UNKNOWN, unknownMsg);
        }
    }

    private Throwable getThrowable(Throwable e) {
        Throwable throwable = e;
        while (throwable.getCause() != null) {
            e = throwable;
            throwable = throwable.getCause();
        }
        return e;
    }

    private void resultError(ResultException e) {
        if (e.getErrCode() == ApiException.RE_LOGIN) {
            try {
                Context currentActivity = ActivityUtil.getCurrentActivity();
                if (currentActivity != null) {
                    Intent intent = new Intent(currentActivity, Class.forName("com.kingyon.onehouse.uis.activities.LoginActivity"));
                    currentActivity.startActivity(intent);
                    Toast.makeText(currentActivity, "请重新登录", Toast.LENGTH_SHORT).show();
                }
            } catch (Exception e1) {
                e1.printStackTrace();
            }
        }
    }

    /**
     * 错误信息回调
     */
    private void error(Throwable e, int errorCode, String errorMsg) {
        ApiException ex = new ApiException(e, errorCode);
        Logger.d(e);
        ex.setDisplayMessage(errorMsg);
        onResultError(ex);
    }

    /**
     * 服务器返回的错误
     */
    protected abstract void onResultError(ApiException ex);

    @Override
    public void onCompleted() {
    }

}
```

这个类中可以，拿到错误信息，并把特殊的错误拦截到做处理，比如这里我们项目中如果错误代码是1000，则要重新登录。

然后再来看看如何用：

```
import android.content.Context;

import net.arvin.afbaselibrary.nets.BaseNet;
import net.arvin.androidart.App;

import retrofit2.converter.gson.GsonConverterFactory;

/**
 * created by arvin on 17/2/27 20:05
 * email：1035407623@qq.com
 */
public class ArtNet extends BaseNet<ArtApi> {
    private static ArtNet mNet;

    public ArtNet() {
        super();
        converterFactory = GsonConverterFactory.create();
    }

    public static ArtNet getInstance() {
        if (mNet == null) {
            mNet = new ArtNet();
        }
        return mNet;
    }

    @Override
    protected String getBaseUrl() {
        return ArtApi.baseUrl;
    }

    @Override
    protected Context getContext() {
        return App.getInstance();
    }
}
```

继承自BaseNet，然后再看看Api，

```
import net.arvin.androidart.entities.GitHubRepos;

import java.util.List;

import retrofit2.http.GET;
import retrofit2.http.Query;
import rx.Observable;

/**
 * created by arvin on 17/2/27 20:05
 * email：1035407623@qq.com
 */
public interface ArtApi {
    String baseUrl = "https://api.github.com/";

    @GET("users/arvinljw/repos")
    Observable<List<GitHubRepos>> repos(@Query("page") int page,@Query("per_page")int per_page);
}
```
这里是分页查看我的github仓库，最后展示到界面上，

```
ArtNet.getInstance().getApi().repos(page, DEFAULT_SIZE).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())
    .subscribe(new AbsAPICallback<List<GitHubRepos>>() {
        @Override
        protected void onResultError(ApiException ex) {
            showToast(ex.getDisplayMessage());
            refreshComplete(false);
        }

        @Override
        public void onNext(List<GitHubRepos> gitHubRepos) {
            if (page == FIRST_PAGE) {
                mItems.clear();
            }
            mItems.addAll(gitHubRepos);
            refreshComplete(gitHubRepos.size() > 0);
        }
    });
```

都是一些简单的封装，介绍的不好，如果有兴趣就直接看我的demo吧。

## Demo源码

[Retrofit简单的使用](https://github.com/arvinljw/AndroidArt/tree/master/app/src/main/java/net/arvin/androidart/retrofit)

















