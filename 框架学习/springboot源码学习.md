[TOC]

# 一、@SpringBootApplication



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





