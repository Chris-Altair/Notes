[TOC]

## 一、测试思考

​	存在一种先写测试再写代码的思路，先构思一下功能、输入输出，编写测试代码，然后写功能代码，跑测试，提交；

​	测试的粒度不好把握，太多了费时间且可能没必要，太少了又失去了测试的意义。

## 二、测试框架testng学习

### 1. 引用依赖

```xml
<dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>7.1.0</version>
            <scope>test</scope>
</dependency>
```

### 2.springboot使用测试方法

​	AbstractTestNGSpringContextTests表示不开启事务

​	AbstractTransactionalTestNGSpringContextTests表示开启事务，默认@Test修饰的方法会回滚数据，除非加上@Rollback(false)

**注：事务与mybatis-plus多数据源冲突**

```java
/**
 * 其余测试类只需继承BaseCase就行
 */
@ActiveProfiles("test") //使用application-test.yml配置文件的配置
@SpringBootTest(classes = XXXApplication.class) //项目的启动类
public abstract class BaseCase extends AbstractTestNGSpringContextTests {}
```

### 3.@DataProvider数据提供者

​	DataProvider为@Test测试方法提供数据，测试方法可带参数，支持批量测试，要保证与Test方法参数顺序及类型一致;

​	parallel = true 开启并行执行，默认false；默认会按顺序执行dataProvider提供的测试用例，开启后会并行执行dataProvider提供的测试用例。

```java
@DataProvider(name = "userDataPro")
    public Object[][] userDataPro() {
        return new Object[][]{
                {"Alice", "123456", false},
                {"Pachi", "123456", false}
        };
    }
```

### 4.@Test注解

​	修饰测试方法，该注解属性如下：

#### 	①dataProvider和dataProviderClass

​		dataProvider=@DataProvider.name

​		dataProviderClass=@DataProvider修饰方法所在的类，可省

```java
@Autowired
private UserService userService;
@Test(dataProvider = "userDataPro", dataProviderClass = LoginCaseDataPro.class)
    public void testGetUserByUserNameAndPasswd(String username, String password, boolean isNotNull) {
        TblUserregister user = userService.getUserByUserNameAndPasswd(username, password);
        if (isNotNull) {
            Assert.assertNotNull(user);
        } else {
            Assert.assertNull(user);
        }
    }
```

#### 	②expectedExceptions

​		测试方法是否会抛出预期的异常，如能抛出则pass，否则fail

```java
@Test(expectedExceptions = {ArithmeticException.class})
```

#### 	③dependsOnMethods

​		当前测试方法依赖该属性注入的测试方法，若注入的测试方法失败则skip当前测试方法；若加上alwaysRun=true，则无论注入的测试方法是否成功，都会执行。

```java
//testProcessRealNumbers依赖testAdd、testDivide，只有testAdd和testDivide都测试成功才会执行testProcessRealNumbers，
//否则会跳过testProcessRealNumbers
@Test(dependsOnMethods={"testAdd", "testDivide"}) public void testProcessRealNumbers(){}
```

#### 	④threadPoolSize和invocationCount

​		threadPoolSize=线程池线程数，invocationCount=总共执行次数；

​		**注意：不管threadPoolSize设置多少值，都只执行invocationCount次；若注入dataProvider则每组参数执行invocationCount次**

```java
//该测试方法可在3个线程中并发执行，共被调用10次，执行超过10秒
@Test(threadPoolSize = 3, invocationCount = 10,  timeOut = 10000) public void test1() {...}
//如果注入dataProvider，则以dataProvider整体会被执行，即总共执行4*10个测试用例            
@Test(threadPoolSize = 3, invocationCount = 10,  timeOut = 10000,dataProvider="有4个") public void test2() {...}
```

#### 	⑤enabled

​		是否启用当前测试方法，默认true,设置false的话不会执行该测试方法，但要注意该方法不能被其他的测试方法所依赖，否则会报错

#### 	⑥timeOut

​		此测试应花费的最大毫秒数

#### 	⑦ignoreMissingDependencies

​		默认false，true则表示忽略依赖的方法，即即便testDivide这个方法被禁用了，下面的测试也能执行
​    	注：alwaysRun是即便失败也能执行，但若方法被禁用的话会报错，ignoreMissingDependencies则是直接忽略了；alwaysRun是针对是否失败，ignoreMissingDependencies为是否忽略

```java
//表示该方法忽略dependsOnMethods引入的依赖
@Test(dependsOnMethods={"testAdd", "testDivide"}, ignoreMissingDependencies = true)
```

### 5.maven测试插件配置

​	mvn install 会在本地maven仓库生成依赖包
​    pom文件依赖设置score：test
​        Maven本身并不是一个单元测试框架，它只是在构建执行到特定生命周期阶段的时候，通过插件来执行JUnit或者TestNG的测试用例。这个插件就是maven-surefire-plugin，也可以称为测试运行器(Test Runner)，它能兼容JUnit 3、JUnit 4以及TestNG。
​        在默认情况下，maven-surefire-plugin的test目标会自动执行测试源码路径（默认为src/test/java/）下所有符合一组命名模式的测试类。这组模式为：

```
**/Test*.java：任何子目录下所有命名以Test开关的Java类。
**/*Test.java：任何子目录下所有命名以Test结尾的Java类。
**/*TestCase.java：任何子目录下所有命名以TestCase结尾的Java类。
```

maven配置testng

```xml
<plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <configuration>
                <suiteXmlFiles>
                    <suiteXmlFile>src/main/resources/testng.xml</suiteXmlFile>
                </suiteXmlFiles>
            </configuration>
        </plugin>
```

### 6.testng.xml配置

​	idea可安装testng插件，直接右键执行testng.xml

​	suite:常用配置：verbose（控制台输出的详细内容等级,0-10级（0无，10最详细），不在报告显示）

​	parallel（线程级别，默认false）

​	thread-count（线程池数量，默认5）
​        test:preserve-order=true设置顺序执行
​        groups:<run>表示运行的组，必须配置clasess，否则不会执行；设置组后，只会执行组内的test方法
​        classes：<class><method>

```xml
<!DOCTYPE suite SYSTEM "http://testng.org/testng-1.0.dtd">
<!--    parallel是否在不同的线程并行进行测试，要与thread-count配套使用-->
<!--    parallel取值：false，methods，tests，classes，instances。默认false-->
<!--    tests表示tests内的方法在同一线程，一个test对应一个线程，故名tests为单位-->
<!--    thread-count与parallel配套使用，线程池的大小，决定并行线程数量；整数默认5-->
<!--    thread-count =3  线程数为3，可同时执行3个case-->
<suite name="First suite" verbose="1" parallel="tests">
<!--    <listeners>-->
<!--        <listener class-name="cn.edu.csc.csc3.app.base.listener.TestListener"></listener>-->
<!--    </listeners>-->
    <test name="testOldUser" parallel="methods">
        <!--查找包下的所有包含testNG annotation的类或方法进行测试-->
<!--        <packages>-->
<!--            <package name="cn.edu.csc.csc3.app.base.test.*">-->
<!--                <exclude name="testPage"></exclude>-->
<!--                <exclude name="aaa"></exclude>-->
<!--                <exclude name="GUserServiceCase"></exclude>-->
<!--            </package>-->
<!--        </packages>-->
        <!-- 执行测试类的测试方法,默认全执行，include包括，exclude排除 -->
        <classes>
<!--            classes下必须要有class，否则不会执行任何测试-->
            <class name="cn.edu.csc.csc3.app.base.test.TestOldUserCase">
                <methods>
<!--                    <exclude name="testGetOldUserByUserNameAndPasswd"></exclude>-->
                </methods>
            </class>
        </classes>
    </test>
    <test name="testUtil" parallel="classes">
        <!-- 设置运行组内的方法，前提是需要存在classes的组，否则所有方法不被运行 -->
        <groups>
            <run>
                <!-- 加了组后，只会执行组内的test方法 -->
                <include name="testUtilAdd"/>
                <include name="testUtilDiv"/>
            </run>
        </groups>
        <!-- 执行测试类的测试方法,默认全执行，include包括，exclude排除 -->
        <classes>
            <class name="cn.edu.csc.csc3.app.base.test.TestNGDependsOnMethodsExample">
                <methods>
                    <exclude name="testProcessRealNumbers"></exclude>
                    <exclude name="testProcessEvenNumbers"></exclude>
                </methods>
            </class>
            <class name="cn.edu.csc.csc3.app.base.test.common.RedisCase"/>
        </classes>
    </test>
</suite>

```

### 7.testng思考总结

默认带返回值的测试方法在xml里不会执行，可能要设置allow-return-values
        观点；
            1.单元测试是要隔绝对数据库的依赖的，通常都要使用Mock技术，接口+mock
            2.接口+Mock来做单元测试，最大化覆盖，剩下的一点数据库访问就用集成测试
        疑问：
            不少逻辑是写在sql里的或者service比较简单dao复杂，mock了dao，测试还有什么意义？
            如果用before、after,首先多线程执行测试存在线程安全问题，再者
        解答：
            单元测试应该屏蔽环境，中间件等一切外部干扰，只测试代码逻辑分支是否符合预期
            单元测试不应该启动spring，因为太慢，单元测试应该频繁执行；而且如果启动spring,mock也失去了意义
        service使用mock，util使用powermock
        以service、util为主

## 三、mock框架学习

​	模拟方法，可将集成测试降级为单元测试

注：dependsOnMethods与mock方向不同，设当前方法为method1,dependsOnMethods是依赖的方法无误method1会执行，否则会skip;
            mock是模拟依赖的方法，保证mock的方法如预期,method1一定会执行
            dependsOnMethods更像是集成测试，测试method1及依赖的方法整体功能是否符合预期；mock是单元测试，只测试method1方法本身是否符合预期，不测试method1依赖的方法。
        问题1：dao注入了mock,但service调用dao mock后的方法没有执行，service内没有注入mock的dao
        问题2：(不使用mockBean)：mockBean修饰的bean A，如果其他的spring bean类内注入了也A会报1 but 2错误，即单例bean发现多个的问题（但有的注入没报这个问题），
        @mock模拟（支持多个），@InjectMocks注入模拟，在初始化方法中执行MockitoAnnotations.initMocks(this)，然后只需写模拟规则即可
    坑：1.使用mybatis-plus mock参数为queryWrapper，queryWrapper如果使用lambda()会报错，应该先执行TableInfoHelper.initTableInfo(null, <entity>.class)初始化表
            2.@InjectMocks必须修饰class，不能修饰接口
            3. when().thenReturn
                powermockito:可支持mock静态、私有及构造方法，注意mock和powermock版本有要求
                坑：1.测试类必须继承PowerMockTestCase
                        2.类上必须添加@PrepareForTest(<StaticUtil>.class)
                        3.必须先初始化，以静态方法为例PowerMockito.mockStatic(<StaticUtil>.class);
                        4.方法的参数必须使用mock的方法不能自己new，正例：PowerMockito.when(JsonUtil.toJson(any())).thenReturn("fuck")

```java
import com.baomidou.mybatisplus.core.metadata.TableInfoHelper;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.Mockito;
import org.mockito.MockitoAnnotations;
import org.powermock.api.mockito.PowerMockito;
import org.powermock.core.classloader.annotations.PrepareForTest;
import org.powermock.modules.testng.PowerMockTestCase;
import org.testng.Assert;
import org.testng.annotations.BeforeClass;
import org.testng.annotations.Test;

import java.util.Arrays;
import java.util.List;

import static org.mockito.ArgumentMatchers.any;



@PrepareForTest(JsonUtil.class)
public class LoginServiceCase extends PowerMockTestCase {
    @Mock
    private TUserMapper tUserMapper;
    @Mock
    private GUserMapper gUserMapper;
    @Mock
    private SUserMapper sUserMapper;
    @InjectMocks
    private LoginServiceImpl loginService;

    @BeforeClass
    public void init() {
        MockitoAnnotations.initMocks(this);
        //解决mock无法模拟mybatis-plus框架queryWrapper的lambda()缓存表信息为空的问题
        TableInfoHelper.initTableInfo(null, TUser.class);
        TableInfoHelper.initTableInfo(null, SUser.class);
        TableInfoHelper.initTableInfo(null, GUser.class);
        PowerMockito.mockStatic(JsonUtil.class);
    }

    /**
     * 获取用户名service测试
     *
     * @param loginType
     * @param verifyType
     * @param verifyWay
     * @param resultLength
     */
    @Test(groups = "forgetAccountGroup",
            dataProvider = "getUsernamesByVerifyWayDataPro", dataProviderClass = LoginCaseDataPro.class)
    public void getUsernamesByVerifyWay(String loginType, String verifyType, String verifyWay, int resultLength) {
        PowerMockito.when(JsonUtil.toJson(any())).thenAnswer(invocationOnMock -> {
            System.out.println("mock fuck");
            return "fuck";
        });
//        Assert.assertEquals(JsonUtil.toJson("aaa"), "fuck");
        Mockito.when(tUserMapper.selectList(any(QueryWrapper.class)))
                .thenReturn(Arrays.asList(new TUser().setUserName("Alice"), new TUser().setUserName("Pachi")));
        Mockito.when(sUserMapper.selectList(any(QueryWrapper.class)))
                .thenReturn(Arrays.asList(new SUser().setUserName("Alice"), new SUser().setUserName("Pachi")));
        Mockito.when(gUserMapper.selectList(any(QueryWrapper.class)))
                .thenReturn(Arrays.asList(new GUser().setUserName("Alice"), new GUser().setUserName("Pachi")));
        List<String> usernames = loginService.getUsernamesByVerifyWay(loginType, verifyType, verifyWay);
        System.out.println("usernames = " + usernames);
        Assert.assertEquals(usernames.size(), 2);
    }
}
```

## 四、集成测试

​    1. 使用h2内存数据库模拟数据，设置oracle模式

```properties
spring.datasource.url = jdbc:h2:mem:test;Mode=Oracle;DB_CLOSE_DELAY=-1 #创建test数据库，mem:数据存内存，oracle模式
spring.datasource.username # 可以没有
spring.datasource.password # 可以没有
spring.datasource.schema = classpath:db/schema.sql #初始化表结构
spring.datasource.data = classpath:db/data.sql #初始化表数据
```

	2. h2支持事务， AbstractTestNGSpringContextTests不启用事务AbstractTransactionalTestNGSpringContextTests开启事务，
        继承后默认开启事务，也可以使用@Rollback来控制测试方法是否启用事务，@Rollback(true)开启事务
        **注：事务只限于@test方法级别，@beforeClass插入的数据，永久插入,目前只能在@AfterClass手动删除**
 	3.  启用不同的配置文件 @ActiveProfiles("testng") 启用application-testng.yml配置文件
             开启springtest环境@SpringBootTest(classes = XXXApplication.class)

## 附录：pom配置

```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.mockito</groupId>
                    <artifactId>mockito-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.powermock</groupId>
            <artifactId>powermock-api-mockito2</artifactId>
            <version>2.0.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.powermock</groupId>
            <artifactId>powermock-module-testng</artifactId>
            <version>2.0.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>2.23.0</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>7.1.0</version>
            <scope>test</scope>
        </dependency>
```

