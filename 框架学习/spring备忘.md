抽象父类中可以使用@Autowired注入，然后子类注册为bean，测试可以成功调用

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

