maven动态切换springboot配置文件

1. pom.xml配置

```xml
<profiles>
        <profile>
            <!-- 生产环境 -->
            <id>prod</id>
            <properties>
                <profiles.active>prod</profiles.active>
            </properties>
        </profile>
        <profile>
            <!-- 本地开发环境 -->
            <id>dev</id>
            <properties>
                <profiles.active>dev</profiles.active>
            </properties>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
        <profile>
            <!-- 测试环境 -->
            <id>test</id>
            <properties>
                <profiles.active>test</profiles.active>
            </properties>
        </profile>
    </profiles>
```

2. yml配置

```yml
spring:
  profiles:
    active: @profiles.active@ # 指定当前环境
```

3. 命令打包

```bash
mvn clean package -P dev #指定dev配置环境
```



