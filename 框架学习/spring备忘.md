抽象父类中可以使用@Autowired注入，然后子类注册为bean，测试可以成功调用

（之前一直是使用ApplicationContext做静态调用，这种方法能更好的抽象）

```java
public abstract class Super{
    @Autowired
    public DbService dbService;
    
    public void test(){
        dbService.xxx();
    }
}
@Component
public class A extend Super{
    public void test1(){
        test();//可以成功调用
    }
}
```

BeanCopier 高效bean复制工具

```java
//其实beanCopierMap使用一些本地缓存工具更好
public static Map<String, BeanCopier> beanCopierMap = new HashMap<>();
/**
     * 对象属性拷贝，仅拷贝【属性名及类型一致】的属性
     * 适用场景：拷贝到空对象，生成新类型对象
     * <p>
     * 缺陷：null值会覆盖原先的值
     * 优点：拷贝效率高
     */
    public static <T> T copy(Object source, T target) {
        String key = generateKey(source.getClass(), target.getClass());
        BeanCopier copier = null;
        if (!beanCopierMap.containsKey(key)) {
            copier = BeanCopier.create(source.getClass(), target.getClass(), false);
            beanCopierMap.put(key, copier);
        } else {
            copier = beanCopierMap.get(key);
        }
        copier.copy(source, target, null);
        return target;
    }
```

------

spring切面只能作用于spring bean的方法，不能识别普通方法

spring切面作用的方法不是原对象的方法，而是代理后的方法，这意味着如果存在嵌套调用情况，内部的方法不会被切面识别到的

可参考官网，官网的图描述的很清楚：https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop

```java
public class SimplePojo implements Pojo {

    public void foo() {
        // this next method invocation is a direct call on the 'this' reference
        this.bar();//<=>bar(); 这种调用的是原对象不是代理，所以这种不会被切到
    }

    public void bar() {
        // some logic...
    }
}

```

------

向同事学到了一手关于spring自动注入map的方式，还有这种操作

```java
public abstract class BaseService {
}

@Component("test_service1")
public class Service1 extends BaseService {
}

@Component("test_service2")
public class Service2 extends BaseService {
}

@Component
public class CommonService {
    //Map会自动注入进去,map = {"test_service1":service1,"test_service2":service2}
    @Resource
    private Map<String, BaseService> map;
}
```





