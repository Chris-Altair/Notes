[TOC]

# @SpringBootApplication



## 1. @ComponentScan

组件扫描（要么默认启动类所在包，要么全手配）

- 如果你的其他包层次结构位于使用@SpringBootApplication标注主应用程序下方，则隐式组件扫描将自动涵盖。也就是说，不要明确标注@ComponentScan，Spring Boot会自动搜索当前应用主入口目录及其下方子目录。
- 如果其他包中的bean /组件不在当前主包路径下面，，则应手动使用@ComponentScan 添加（**适合自定义jar包引用，比如公共服务抽取**）
- 如果使用了@ComponentScan ，那么Spring Boot就全部依赖你的定义，如果定义出错，会出现autowired时出错，报a bean of type that could not be found错误。



## 2. @SpringBootConfiguration

@AliasFor可配置别名

* 配置注解内属性互为别名
* 配置跨注解属性别名

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration(proxyBeanMethods = false)
public @interface SpringBootConfiguration {
    /**
    * 配置SpringBootConfiguration#proxyBeanMethods为Configuration#proxyBeanMethods别名
    * 类似于SpringBootConfiguration“继承”了Configuration
    */
	@AliasFor(annotation = Configuration.class)
	boolean proxyBeanMethods() default true;
}
```

@Inherited 元注解 可修饰注解，子类可继承父类的注解（**只适用于类注解！**）

```java
@Target(value = ElementType.TYPE)
    @Retention(value = RetentionPolicy.RUNTIME)
    @Inherited // 声明注解具有继承性
    @interface AInherited {
        String value() default "";
    }

    @Target(value = ElementType.TYPE)
    @Retention(value = RetentionPolicy.RUNTIME)
    @Inherited // 声明注解具有继承性
    @interface BInherited {
        String value() default "";
    }

    @Target(value = ElementType.TYPE)
    @Retention(value = RetentionPolicy.RUNTIME)
            // 未声明注解具有继承性
    @interface CInherited {
        String value() default "";
    }

    @AInherited("父类的AInherited")
    @BInherited("父类的BInherited")
    @CInherited("父类的CInherited")
    class SuperClass {}

    @BInherited("子类的BInherited")
    class ChildClass extends SuperClass {}

    public static void main(String[] args) {
        Annotation[] annotations = ChildClass.class.getAnnotations();
        System.out.println(Arrays.toString(annotations));
        // output: [@annotations.InheritedTest1$AInherited(value=父类的AInherited), @annotations.InheritedTest1$BInherited(value=子类的BInherited)]
    }
```

属性和方法注解的继承，与类注解的继承完全不同，与元注解Inherited毫无关系，忠实于方法/属性本身的继承。

```java
@Target(value = {ElementType.METHOD, ElementType.FIELD})
    @Retention(value = RetentionPolicy.RUNTIME)
    @interface DESC {
        String value() default "";
    }

    class SuperClass {
        @DESC("父类方法foo")
        public void foo() {}
        @DESC("父类方法bar")
        public void bar(){}
        @DESC("父类的属性")
        public String field;
    }

    class ChildClass extends SuperClass {
        @Override
        public void foo() {
            super.foo();
        }
    }

    public static void main(String[] args) throws NoSuchMethodException, NoSuchFieldException {
        Method foo = ChildClass.class.getMethod("foo");
        System.out.println(Arrays.toString(foo.getAnnotations()));
        // output: []
        // 子类ChildClass重写了父类方法foo,并且@Override注解只在源码阶段保留，所以没有任何注解

        Method bar = ChildClass.class.getMethod("bar");
        System.out.println(Arrays.toString(bar.getAnnotations()));
        // output: [@annotations.InheritedTest$DESC(value=父类方法bar)]
        // bar方法未被子类重写，从父类继承到了原本注解

        Field field = ChildClass.class.getField("field");
        System.out.println(Arrays.toString(field.getAnnotations()));
    }
    // output: [@annotations.InheritedTest$DESC(value=父类的属性)]
    // 解释同上
```

参考：https://www.jianshu.com/p/a848655d478e

# restTemplate消息转换器设计

restTemplate的消息转换器是存放在restTemplate.messageConverters列表里，默认构造函数会往里加入一堆消息转换器

```java
public class RestTemplate extends InterceptingHttpAccessor implements RestOperations {
    ...
    private final List<HttpMessageConverter<?>> messageConverters;
    public RestTemplate() {
        this.messageConverters = new ArrayList();
        this.errorHandler = new DefaultResponseErrorHandler();
        this.headersExtractor = new RestTemplate.HeadersExtractor();
        this.messageConverters.add(new ByteArrayHttpMessageConverter());
        this.messageConverters.add(new StringHttpMessageConverter());
        this.messageConverters.add(new ResourceHttpMessageConverter(false));
	    ...
    }
}
```

我们也可通过注册bean的方式向列表里加入自己的消息转换器：

```java
@Bean
    RestTemplate restTemplate(ClientHttpRequestFactory clientHttpRequestFactory) {
        RestTemplate restTemplate = new RestTemplate(clientHttpRequestFactory);
        List<HttpMessageConverter<?>> converterList = restTemplate.getMessageConverters();
        //使用add方法添加自定义的消息转换器
        converterList.add(xxx)
    }
```

**每种解析器可解析该解析器支持的MediaType类型的消息，若response header的Content-Type满足该转换器的MediaType，则该转换器可处理这种消息**（注意**调用解析器是按顺序的，实际不一定会由该转换器执行**，下面会解释）

可参考官方文档：https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#spring-integration

（ps：有问题看官方文档往往比在网上瞎找强，官方文档都写的很清晰）

比如StringHttpMessageConverter支持 MediaType.TEXT_PLAIN 和 MediaType.ALL

```java
public class StringHttpMessageConverter extends AbstractHttpMessageConverter<String> {
    public static final Charset DEFAULT_CHARSET;
    @Nullable
    private volatile List<Charset> availableCharsets;
    private boolean writeAcceptCharset;

    public StringHttpMessageConverter() {
        this(DEFAULT_CHARSET);
    }
	//比如StringHttpMessageConverter支持 MediaType.TEXT_PLAIN 和 MediaType.ALL
    public StringHttpMessageConverter(Charset defaultCharset) {
        super(defaultCharset, new MediaType[]{MediaType.TEXT_PLAIN, MediaType.ALL});
        this.writeAcceptCharset = false;
    }
```

那么spring是如何使用消息解析器解析http response body呢？，答案是HttpMessageConverterExtractor.extractData(response)

```java
public class HttpMessageConverterExtractor<T> implements ResponseExtractor<T> {
    private final Type responseType;
    @Nullable
    private final Class<T> responseClass;
    private final List<HttpMessageConverter<?>> messageConverters;
    
    ...
    
    @Nullable
    protected MediaType getContentType(ClientHttpResponse response) {
        MediaType contentType = response.getHeaders().getContentType();
        if (contentType == null) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("No content-type, using 'application/octet-stream'");
            }

            contentType = MediaType.APPLICATION_OCTET_STREAM;
        }

        return contentType;
    }
    /**
     * 该方法就是解析并转换response body的
    */
    public T extractData(ClientHttpResponse response) throws IOException {
        MessageBodyClientHttpResponseWrapper responseWrapper = new MessageBodyClientHttpResponseWrapper(response);
        if (responseWrapper.hasMessageBody() && !responseWrapper.hasEmptyMessageBody()) {
            //获取的contentType就是response header里的content-type，为空则默认APPLICATION_OCTET_STREAM
            MediaType contentType = this.getContentType(responseWrapper);

            try {
                //messageConverters其实就是restTemplate.messageConverters
                Iterator var4 = this.messageConverters.iterator();
			   //使用迭代器遍历，这就说明了其实messageConverters的选择是有顺序的
                while(var4.hasNext()) {
                    HttpMessageConverter<?> messageConverter = (HttpMessageConverter)var4.next();
                    if (messageConverter instanceof GenericHttpMessageConverter) {
                        GenericHttpMessageConverter<?> genericMessageConverter = (GenericHttpMessageConverter)messageConverter;
                        //判断该messageConverter是否能处理该contentType
                        if (genericMessageConverter.canRead(this.responseType, (Class)null, contentType)) {
                            if (this.logger.isDebugEnabled()) {
                                ResolvableType resolvableType = ResolvableType.forType(this.responseType);
                                this.logger.debug("Reading to [" + resolvableType + "]");
                            }
						  //read方法其实就是解析并转换消息的
                            return genericMessageConverter.read(this.responseType, (Class)null, responseWrapper);
                        }
                    }

                    if (this.responseClass != null && messageConverter.canRead(this.responseClass, contentType)) {
                        if (this.logger.isDebugEnabled()) {
                            String className = this.responseClass.getName();
                            this.logger.debug("Reading to [" + className + "] as \"" + contentType + "\"");
                        }

                        return messageConverter.read(this.responseClass, responseWrapper);
                    }
                }
            } catch (HttpMessageNotReadableException | IOException var8) {
                throw new RestClientException("Error while extracting response for type [" + this.responseType + "] and content type [" + contentType + "]", var8);
            }

            throw new RestClientException("Could not extract response: no suitable HttpMessageConverter found for response type [" + this.responseType + "] and content type [" + contentType + "]");
        } else {
            return null;
        }
    }
	...
}
```

**实际问题**

之所以看这个源码是因为项目对接三方的接口有些出现了中文乱码的问题，项目的转换器是将第一位的String转换器的默认编码由ISO_8859_1改成UTF_8，然后最后add一个fastjson转换器（项目用的），对方的接口返回的response content-type=text/plain;charset=ISO-8859-1，满足第二位的string转换器处理，导致乱码（其实还原String转换器的默认编码也会乱码。。。）。

经过测试发现使用fastjson转换器不会乱码，仔细debug源码一下，发现使用fastjson转换器的response content-type=application/json;charset=utf-8，于是发现可通过设置accept期望应答头控制对面接口的返回格式，于是就避免了乱码

```java
httpHeaders.setAccept(Collections.singletonList(MediaType.APPLICATION_JSON))
```







