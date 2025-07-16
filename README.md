# Spring Boot Template

项目搭建，讲究一套“好用、好维护、扩展方便”的套路，搞过大型项目的人就知道，最怕后期一堆依赖混乱、包乱飞、改个模块牵一发动全身。

## 项目结构怎么设计

按功能划分+分层架构:
```text
/src
  /main
    /java
      /com.example.project
        /common  // 通用工具类、常量、异常定义
        /config  // 所有 Spring 配置类
        /module  // 各业务模块，每个模块再细分 controller/service/mapper
        /model  // DTO / VO / DO
        /framework // 自定义框架扩展，如拦截器、过滤器、注解等
        /resources
        /mapper  // MyBatis XML 文件
        /static  // 前端资源（如果有的话）
    /resources
      /application.yml // 主配置
```
这个结构可扩展性很强，尤其适合团队开发，每个人负责一个 module，互不干扰。

## 配置先行：yml 一步到位
**`application.yml`** 是 SpringBoot 的大脑，建议一开始就设好基本参数，比如端口号、日志级别、数据库连接啥的。下面是一个通用模板：
```yaml
server:
  port:8080

spring:
  datasource:
    url:jdbc:mysql://localhost:3306/demo?useSSL=false&serverTimezone=UTC
    username:root
    password:root

jackson:
  date-format:yyyy-MM-ddHH:mm:ss
  time-zone:GMT+8

mvc:
  pathmatch:
    matching-strategy:ant_path_matcher

mybatis-plus:
  mapper-locations:classpath:/mapper/**/*.xml
  type-aliases-package:com.example.project.model
```

## 通用返回结构

统一返回格式`Result<T>`

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Result<T> {
    private Integer code;
    private String message;
    private T data;

    public static <T> Result<T> ok(T data) {
        return new Result<>(200, "成功", data);
    }

    public static Result<?> fail(String message) {
        return new Result<>(500, message, null);
    }
}
```
配合全局异常处理器:
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public Result<?> handle(Exception e) {
        log.error("异常捕获：", e);
        return Result.fail("系统异常，请稍后重试");
    }

    @ExceptionHandler(CustomException.class)
    public Result<?> handle(CustomException e) {
        return Result.fail(e.getMessage());
    }
}
```
通用响应封装 & 分页支持
```java
@Data
public class PageResult<T> {
    private List<T> records;
    private long total;

    public PageResult(List<T> records, long total) {
        this.records = records;
        this.total = total;
    }
}
```
在 service 层里直接返回
```java
public PageResult<UserDTO> getUsers(int page, int size) {
    Page<User> userPage = userMapper.selectPage(new Page<>(page, size), null);
    List<UserDTO> dtoList = userPage.getRecords().stream()
            .map(user -> convertToDTO(user))
            .collect(Collectors.toList());
    return new PageResult<UserDTO>(dtoList, userPage.getTotal());
}
```
## 常见功能组件，提前集成
1. 统一日志打印用 `Slf4j`，配合 `@ControllerAdvice` 打日志，推荐接入 `logback.xml` 做级别和格式控制。
2. 跨域支持写个配置类，统一允许所有来源（开发阶段），上线前细化规则。
    ```java
    @Configuration
    public class WebMvcConfig implements WebMvcConfigurer {
        @Override 
        public void addCorsMappings(CorsRegistry registry) {
            registry.addMapping("/**")
                    .allowedOrigins("*")
                    .allowedMethods("*");
        }
    }
    ```
3. 请求参数校验用 `@Validated` 搭配 `@NotBlank`、`@Min` 这种注解，再写个异常捕获就完事。
    ```java
    @PostMapping("/user")
    public Result<?> createUser(@RequestBody @Valid UserCreateRequest request) {
        userService.create(request);
        return Result.ok(null);
    }
    ```
## 模块示例：用户模块怎么拆
`/module/user` 模块
```text
/user
  UserController.java
  UserService.java
  UserServiceImpl.java
  UserMapper.java
  User.java
  UserDTO.java
  UserConvert.java
```
结构清晰，各司其职，不容易混。像 `Convert` 类可以用 `MapStruct` 来生成，非常省事。
```java
@Mapper(componentModel = "spring")
public interface UserConvert { 
    UserDTO toDTO(User user);
}
```

## 工具类提前准备

比如：RedisUtils、JwtUtils、DateUtils、SnowflakeIdGenerator 这些推荐提前集成好，能省一堆活。尤其是雪花 ID，在分布式场景下特别有用：
```java
public class IdGenerator {
    private static final Snowflake snowflake = IdUtil.getSnowflake(1, 1);
    public static long nextId() {
        return snowflake.nextId();
    }
}
```
## 开发体验优化
1. `热部署`：接入 `spring-boot-devtools`，开发速度飙升。
2. `Swagger 接口文档`：默认集成 `Knife4j`，比 `Swagger` 好用多了，文档即测试平台。
3. `代码生成器`：`MyBatis-Plus` 提供代码生成器，数据库表一键生成 entity、mapper、xml，适合大规模开发。

## 部署前别忘的事
- 多环境配置分开，dev/test/pro 的 yml 文件分别管理
- 日志路径改成日志文件输出，别再打控制台了
- 尽量加上接口限流，比如通过 Redis + Lua 实现 QPS 限制
- 数据库连接池参数合理调优，Druid 默认值不适合高并发