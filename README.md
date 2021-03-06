Retrofit源码分析笔记
========
Version:2.2.0
https://github.com/square/retrofit/releases/tag/parent-2.2.0

1. 环境配置相关：
    - 发现项目是基于Maven的，于是祭出许久不用的Idea
    - 删掉了部分对源码无用的配置文件，比如website/,CHANGELOG.md等
2. samples分析
    ```java
        public interface GitHub {
            @GET("/repos/{owner}/{repo}/contributors")
            Call<List<Contributor>> contributors(
                    @Path("owner") String owner,
                    @Path("repo") String repo);
        }
        // Create a very simple REST adapter which points the GitHub API.
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(API_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .build();
        // Create an instance of our GitHub API interface.
        GitHub github = retrofit.create(GitHub.class);
        // Create a call instance for looking up Retrofit contributors.
        Call<List<Contributor>> call = github.contributors("square", "retrofit");
        call.enqueue(new Callback<List<Contributor>>() {
            @Override
            public void onResponse(Call<List<Contributor>> call, Response<List<Contributor>> response) {
                for (Contributor contributor : response.body()) {
                    System.out.println(contributor.login + " (" + contributor.contributions + ")");
                }
            }
            @Override
            public void onFailure(Call<List<Contributor>> call, Throwable t) {

            }
        });
    ```
    这是最简单的用法,使用步骤:
    1. 定义一个 API 的 Interface.包含 API 的基本信息(url,method,path,call,返回类型等)
    2. 实例化Retrofit对象,配置 baseUrl 和 ConverterFactory
    3. 通过 retrofit.create 方法创建 API 实例对象
    4. 直接调用具体的 api 方法得到 Call 对象,调用enqueue方法传入回调
    
3. 基本函数调用链
    - Retrofit.Build 建造者模式
    - Retrofit.create() 创建相应Api interface
        - eagerlyValidateMethods 校验解析所有method,即定义的每个api method
            - ServiceMethod.build() 解析interface所有method，每个method的参数封装成ServiceMethod
                - ServiceMethod.createCallAdapter() 每个method寻找匹配的CallAdapter，即返回值转换器
                    - Retrofit.callAdapter() 遍历retrofit配置过的adapter，寻找匹配的那个
                        - Retrofit.nextCallAdapter()
                - ServiceMethod.createResponseConverter() 创建response转换器
                    - Retrofit.responseBodyConverter() 寻找匹配的responseConverter
                        - Retrofit.nextResponseBodyConverter() 
                - ServiceMethod.parseMethodAnnotation() 解析标记方法的每个注解，比如Get,Post等常见注解就是在这里解析
                    - ServiceMethod.parseHttpMethodAndPath() 解析具体的Http注解和path，比如GET
                        - ServiceMethod.parsePathParameters() 解析url包含的路径参数，即{path}，params必须包含@Path
                    - ServiceMethod.parseHeaders() 解析head，通过@Headers判断
                - ServiceMethod.parseParameter() 解析每个param,返回ParameterHandler
                    - ServiceMethod.parseParameterAnnotation() 解析参数的具体注解,常见的有@path,@query等,requestConverter和stringConverter在这会被调用
                        - ParameterHandler 这个类主要是api方法参数的各种具体处理，添加到RequestBuilder
        - Proxy.newProxyInstance() 动态代理，代理整个Api interface，当我们调用api方法时，会调用这个代理的invoke()方法，这是retrofit的核心
            - OkHttpCall OKHttp具体请求网络的地方,拿出初始化传入的OkHttpClient，拿出ServiceMethod.toRequest()的Request，然后怼到一起，调用okhttp的api进行网络请求和回调处理，在parseResponse()中会调用responseConverter
            - CallAdapter.adapt() Android默认的CallAdapter在ExecutorCallAdapterFactory里，返回的是Call<T>,另外提供的有RxJava2CallAdapter,返回值是Observable
4. 关键点的代码查看
    1. 为什么只用定义一个interface，达到直接调用的效果
        利用了Java动态代理机制,和 AOP 一个原理，相关代码在Retrofit.create();
        ```java
            return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[]{service},
                            new InvocationHandler() {
                                private final Platform platform = Platform.get();
                                @Override
                                public Object invoke(Object proxy, Method method, Object[] args)
                                        throws Throwable {
                                    //......
                                    //NOTE-Blanke: 加载这个method，如果设置validateEagerly=true,这里是直接获取serviceMethodCache里的缓存
                                    ServiceMethod<Object, Object> serviceMethod =
                                            (ServiceMethod<Object, Object>) loadServiceMethod(method);
                                    //NOTE-Blanke: 默认http调用OKHttp，这里可以扩展，换成别家的http框架
                                    OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
                                    return serviceMethod.callAdapter.adapt(okHttpCall);
                                }
                            });    
        ```
    2. 平常使用最多的是json，相关转换代码
        以Gson解析为例,相关代码在retrofit2.converter.gson.
        - requestConvert 
            见GsonRequestBodyConverter
            具体的转换代码在ParameterHandler.Body，具体调用链是ServiceMethod.parseParameterAnnotation()->new Body,ServiceMethod.toRequest()->Body.apply();
        - responseConvert
            见GsonResponseBodyConverter
            具体转换代码在ServiceMethod.toResponse().具体调用链是OkHttpCall.parseResponse()->ServiceMethod.toResponse();
    3. 自定义CallAdapter相关
       默认的是ExecutorCallAdapterFactory匿名类，默认的返回类型是Call.
       调用链是Retrofit.create():`serviceMethod.callAdapter.adapt(okHttpCall)`
       RxJava是RxJava2CallAdapter，返回类型是Observable
       自定义demo见ErrorHandlingAdapter
5. 发现的设计模式
    - 建造者模式 出现的Builder类
    - 工厂模式 以Factory结尾的类
    - 代理模式 ExecutorCallbackCall，Retrofit.create()用到了动态代理
    - 策略模式 Converter,CallAdapter等的实现
    - 适配器模式 ErrorHandlingAdapter.MyCallAdapter
    - 迭代器模式 ParameterHandler
    - 外观模式 Retrofit
    