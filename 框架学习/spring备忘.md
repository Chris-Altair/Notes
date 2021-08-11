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

