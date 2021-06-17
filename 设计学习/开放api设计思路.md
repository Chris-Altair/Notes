对接过程中发现的各种开放api接口设计

## 一、认证方面

### 1. oauth2协议

​	标准oauth2授权码模式，先获取code，通过code拿token，后续通过token校验接口，若token过期则调用refresh接口获取新token，refreshToken过期则重新走oauth2流程

​	**优点**：不暴露用户密码（三方无法获取到）

​	**缺点**：流程相对复杂

### 2. login接口获取token

​	先通过username和password调用login接口获取token，开放api通过token校验，token过期则重新调用login接口

​	**优点**：实现相对简单

​	**缺点**：暴露用户名和密码

### 3. 用户信息与api权限分离模式

​	用户登录页面后可查看**apiKey**和**apiToken**，第三方调用开放api通过该key和token验证权限

​	核心思想是**将api的权限与用户名密码分离开**，

​	**优点**：对接时不会暴露用户信息，就算泄露了key和token，刷新token就可解决

​	**缺点**：基本上需要第三方写死token,若token已被刷新，三方不能实时获取刷新后的token

​	当然这个缺点也可通过refresh接口参数为key和token，返回新token来解决

**结论**：综合考虑，就api认证方向考虑，权限分离模式比较好

**备注**：有的token使用head authorization bear方式，也有的直接放到参数里

## 二、幂等性

可通过业务唯一id来控制，比如推单一般都有订单id

也可添加个接口唯一id，将id放到缓存(有效期)中，在缓存中就跳过，也可通过布隆过滤器实现

## 三、防篡改

通过参数加时间戳timestamp及签名sign实现，通过timestamp可保证接口时效性；通过对请求参数和时间戳进行签名或计算摘要，验证sign是否一致来判断

## 四、其他

接口文档的字段一定要精准，如果是同一个字段，就不该起俩名