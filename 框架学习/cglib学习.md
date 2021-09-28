用法参考，写的比较全面，很适合初学：https://frameworks.readthedocs.io/en/latest/spring/aop/cglib.html

### 1.使用示例

```java
public static <T extends RequestEntity> T getProxy(T t, IClient client, ThirdApiMethod apiMethod) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(t.getClass());
        enhancer.setCallback((MethodInterceptor) (o, method, args, proxy) -> {
            if (EXCHANGE.equals(method.getName())) {
                final long beginTime = System.currentTimeMillis();
                //调用三方接口则统计耗时
                final Object result = proxy.invokeSuper(o, args);
                final long endTime = System.currentTimeMillis();
                log.info("{}->{} 三方接口调用耗时:{}ms", client.getClass().getSimpleName(), apiMethod.name(), (endTime - beginTime));
                return result;
            } else {
                return proxy.invokeSuper(o, args);
            }
        });
        //需要注意:
        //1、需要注意create()方法只是创建了代理对象，原对象t内的属性并未赋值给代理对象，所以需要执行深拷贝
        //2、注意多次执行该方法若t的类型一致，则只会生成一次代理类，不会重复创建代理类（当然每次都会创建新的代理对象）
        return BeanCopyUtil.copy(t, (T) enhancer.create());
    }
```

### 2.源码分析

在main方法执行代理前，加上下面的语句可将代理类输出到指的路径中

```java
//将生成的代理对象输出到D盘
System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:\\");
```

观察路径，可见生成了3个类，注：*HttpRequestEntity*即为目标类（也可叫被代理类）

```bash
#生成类的名字规则是：被代理classname + "$$"+classgeneratorname+"ByCGLIB"+"$$"+key的hashcode

#代理类FastClass类
HttpRequestEntity$$EnhancerByCGLIB$$57735f41$$FastClassByCGLIB$$5a00d218.class
#代理类，继承了HttpRequestEntity类
HttpRequestEntity$$EnhancerByCGLIB$$57735f41.class
#目标类FastClass类
HttpRequestEntity$$FastClassByCGLIB$$50faa27a.class
```

先看示例代码中的*proxy.invokeSuper*方法

```java
public Object invokeSuper(Object obj, Object[] args) throws Throwable {
        try {
            //初始化，创建了目标类及代理类的FastClass对象
            init();
            FastClassInfo fci = fastClassInfo;
            // 这里将直接调用代理类的CGLIB$exchange$11方法，而不是通过反射调用
            // fci.f2：代理类的FastClass对象，fci.i2为CGLIB$exchange$11方法对应的索引，obj为当前的代理类对象，args为exchange方法的参数列表
            return fci.f2.invoke(fci.i2, obj, args);
        } catch (InvocationTargetException e) {
            throw e.getTargetException();
        }
    }
private static class FastClassInfo
    {
        //目标类FastClass对象
        FastClass f1;
        //代理类FastClass对象
        FastClass f2;
        int i1;
        int i2;
    }

private void init() {
		/*
		 * Using a volatile invariant allows us to initialize the FastClass and
		 * method index pairs atomically.
		 *
		 * Double-checked locking is safe with volatile in Java 5.  Before 1.5 this
		 * code could allow fastClassInfo to be instantiated more than once, which
		 * appears to be benign.
		 */
		if (fastClassInfo == null) {
			synchronized (initLock) {
				if (fastClassInfo == null) {
					CreateInfo ci = createInfo;

					FastClassInfo fci = new FastClassInfo();
                      // 创建目标类的FastClass对象
					fci.f1 = helper(ci, ci.c1);
                      // 创建代理类的FastClass对象
					fci.f2 = helper(ci, ci.c2);
                      // 获取【目标类的FastClass对象】的exchange方法的索引
					fci.i1 = fci.f1.getIndex(sig1);
                      // 获取【代理类的FastClass对象】的exchange方法的索引
                      // proxy.invokeSuper实际就是调的这个索引对应的方法
					fci.i2 = fci.f2.getIndex(sig2);
					fastClassInfo = fci;
					createInfo = null;
				}
			}
		}
	}
```

这是***HttpRequestEntity$$EnhancerByCGLIB$$57735f41$$FastClassByCGLIB$$5a00d218***类的invoke部分方法

```java
public Object invoke(int var1, Object var2, Object[] var3) throws InvocationTargetException {
        //var10000其实就是生成的代理对象HttpRequestEntity$$EnhancerByCGLIB$$57735f41
        //你看57735f41就是代理对象的后缀
        57735f41 var10000 = (57735f41)var2;
        int var10001 = var1;

        try {
            //通过方法索引直接找到方法，避免了反射的开销，这就是为什么cglib比反射更快的原因
            switch(var10001) {
            ...
            case 15:
                var10000.setUrl((String)var3[0]);
                //void方法返回null
                return null;
            ...
            case 42:
                //这就是invokeSuper实际调用的方法
                return var10000.CGLIB$exchange$11((RestTemplate)var3[0]);
            ...
            }
        } catch (Throwable var4) {
            throw new InvocationTargetException(var4);
        }
        throw new IllegalArgumentException("Cannot find matching method/constructor");
    }
```

现在我们已经知道*MethodProxy.invokeSuper*是调的是CGLIB$exchange$11方法，那这个方法是在哪里呢？

答案是该方法位于代理类中，代理类继承了目标类，代理方法最终会调用父类的exchange方法

```java
/**
* 代理类，继承了目标类
*/
public class HttpRequestEntity$$EnhancerByCGLIB$$57735f41 extends HttpRequestEntity implements Factory {
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    //我们在enhancer.setCallback的MethodInterceptor对象
    private MethodInterceptor CGLIB$CALLBACK_0;
    private static Object CGLIB$CALLBACK_FILTER;
    private static final Method CGLIB$equals$0$Method;
    private static final MethodProxy CGLIB$equals$0$Proxy;
    private static final Object[] CGLIB$emptyArgs;
    ...
    //目标类的exchange方法对象
    private static final Method CGLIB$exchange$11$Method;
    //代理类的exchange方法对象
    private static final MethodProxy CGLIB$exchange$11$Proxy;
    private static final Method CGLIB$getInnerParams$12$Method;
    private static final MethodProxy CGLIB$getInnerParams$12$Proxy;
    ...
    
    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        Class var0 = Class.forName("cn.wangdian.config.http.HttpRequestEntity$$EnhancerByCGLIB$$57735f41");
        Class var1;
        ...
        CGLIB$exchange$11$Method = var10000[11];
        //创建了方法代理
        CGLIB$exchange$11$Proxy = MethodProxy.create(var1, var0, "(Lorg/springframework/web/client/RestTemplate;)Lorg/springframework/http/ResponseEntity;", "exchange", "CGLIB$exchange$11");
    }
    
    ...
        
    final ResponseEntity CGLIB$exchange$11(RestTemplate var1) {
        //该方法就是最终调用的方法
        return super.exchange(var1);
    }

    public final ResponseEntity exchange(RestTemplate var1) {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }
	    //intercept就是我们在enhancer.setCallback的
        return var10000 != null ? (ResponseEntity)var10000.intercept(this, CGLIB$exchange$11$Method, new Object[]{var1}, CGLIB$exchange$11$Proxy) : super.exchange(var1);
    }
```

总结一下，invokeSuper方法会先初始化目标类和代理类的FastClass对象，并设置目标方法的索引，然后在代理FastClass对象中根据索引直接找到代理对象的代理方法（这就是为什么cglib比反射快的原因），代理对象继承了原对象，最终还是会调到原对象的方法

proxyFastClass -> proxyClass -> targetClass

参考：https://zhuanlan.zhihu.com/p/106069224