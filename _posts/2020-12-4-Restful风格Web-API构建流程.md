---
layout:     post
title:      Restful风格Web API构建流程
subtitle:   一套属于自己的脚手架。
date:       2020-12-4
author:     BY, Tanyongfeng 
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - SpringBoot	
    - 后端开发
    - Java
---

# Restful风格Web API构建流程

## 零、技术栈：

|   框架类型   |    框架名称     |
| :----------: | :-------------: |
|   基本框架   |   SpringBoot    |
|    持久层    |   Spring JPA    |
| 关系型数据库 |      Mysql      |
|  缓存数据库  |      Redis      |
|   渲染引擎   |    Thymeleaf    |
| 权限控制框架 | Spring Security |

## 一、编写Restful风格接口类

### 1.添加ResponseUtil类（用来返回Resuful风格Json数据）

```java
/**
 * @Describe: 类描述
 * @Author: tyf
 * @CreateTime: 2021/11/15
 **/
public class ResponseUtil {
    public static <T> void restResponse(HttpServletResponse resp, HttpStatus status, RestResult<T> responseVo) throws IOException {
        resp.setStatus(status.value());
        resp.setContentType(MediaType.APPLICATION_JSON.toString());
        resp.getOutputStream().write(new ObjectMapper().writeValueAsBytes(responseVo));
    }
}
```

### 2.添加`ResultEnum`枚举类（用来返回服务状态）

```java
/**
 * @Describe: 类描述
 * @Author: tyf
 * @CreateTime: 2021/11/15
 **/
@Getter
public enum ResultEnum {
    INTERNAL_WRONG(500, "内部错误!");

    private final Integer resultCode;
    private final String resultMsg;

    ResultEnum(Integer resultCode, String resultMsg) {
        this.resultCode = resultCode;
        this.resultMsg = resultMsg;
    }
} 
```

### 2.添加`RestResult`类（Restful风格返回体）

```java
@Data
@JsonInclude(JsonInclude.Include.NON_NULL)
public class RestResult<T> {

    private Integer code;
    private String message;
    private T data;
    private HttpStatus httpStatus;

    public RestResult(Integer code, String message) {
        this.code = code;
        this.message = message;
    }

    public RestResult(Integer code, T data) {
        this.code = code;
        this.data = data;
    }

    public RestResult(HttpStatus status, T data){
        this.httpStatus = status;
        this.data = data;
    }

    public static <T> RestResult<T> success(Integer code, T data) {
        return new RestResult<>(code, data);
    }

    public static <T> RestResult<T> success(T data) {
        return new RestResult<>(200, data);
    }

    public static <T> RestResult<T> success(Integer code, String message) {
        return new RestResult<>(code, message);
    }

    public static <T> RestResult<T> success(ResultEnum resultEnum) {
        return new RestResult<>(resultEnum.getResultCode(), resultEnum.getResultMsg());
    }

    public static <T> RestResult<T> error(Integer code, String message) {
        return new RestResult<>(code, message);
    }

    public static <T> RestResult<T> error(Integer code, T data) {
        return new RestResult<>(code, data);
    }

    public static <T> RestResult<T> error(String message) {
        return new RestResult<>(500, message);
    }

    public static <T> RestResult<T> error(ResultEnum resultEnum) {
        return new RestResult<>(resultEnum.getResultCode(), resultEnum.getResultMsg());
    }
    public static <T> RestResult<T> success(HttpStatus status, T data){
        return new RestResult<>(status, data);
    }
}
```



## 二、 引入相关依赖以及配置基本环境

### 1. 编辑Maven配置文件`pom.xml`

- 添加父文件项目约束

  ```xml
  <parent>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-parent</artifactId>
       <version>2.5.6</version>
       <relativePath/> <!-- lookup parent from repository -->
  </parent>
  ```

- 添加依赖

  ```xml
      <dependencies>
          <!--引入jpa依赖-->
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-data-jpa</artifactId>
          </dependency>
          <!--引入jdbc驱动器-->
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-jdbc</artifactId>
          </dependency>
          <!--引入Spring Security-->
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-security</artifactId>
          </dependency>
          <!--引入渲染引擎-->
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-thymeleaf</artifactId>
          </dependency>
          <!--基础web框架-->
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-web</artifactId>
          </dependency>
          <!--渲染引擎结合前端完成操作-->
          <dependency>
              <groupId>org.thymeleaf.extras</groupId>
              <artifactId>thymeleaf-extras-springsecurity5</artifactId>
          </dependency>
          <!--待选：Excel处理-->
          <dependency>
              <groupId>org.apache.poi</groupId>
              <artifactId>poi</artifactId>
              <version>5.0.0</version>
          </dependency>
          <!--热更新-->
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-devtools</artifactId>
              <scope>runtime</scope>
              <optional>true</optional>
          </dependency>
          <!--连接器-->
          <dependency>
              <groupId>mysql</groupId>
              <artifactId>mysql-connector-java</artifactId>
              <scope>runtime</scope>
          </dependency>
          <!--类管理模块-->
          <dependency>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
              <optional>true</optional>
          </dependency>
          <!--测试框架-->
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-test</artifactId>
              <scope>test</scope>
          </dependency>
      </dependencies>
  ```

### 2. 配置项目配置文件`application.yml`

```yaml
spring:
  imageProxy: https://pic1.xuehuaimg.com/proxy/  #配置图床
  datasource:							 #配置数据源
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/sms?characterEncoding=utf8&serverTimezone=Asia/Shanghai
    username: sms
    password: 123456

  jpa:								#配置jpa
    hibernate:
      ddl-auto: update #打开项目自动更新
    show-sql: true	   #显示sql调用语句，建议produce阶段关闭
    database-platform: org.hibernate.dialect.MySQL8Dialect
server:
  port: 8083				#配置端口号
```

### 3.（可选）配置SpringSecurity

#### 3.1 配置文件

```java
/**
 * @Describe: 类描述
 * @Author: tyf
 * @CreateTime: 2021/11/15
 **/
@Configuration
@EnableWebSecurity
/*开启方法权限认证*/
@EnableGlobalMethodSecurity(securedEnabled=true)
public class SecurityConfigurer extends WebSecurityConfigurerAdapter {

//    private final TigerLogoutSuccessHandler logoutSuccessHandler = new TigerLogoutSuccessHandler("/login");

    public static final String[] NO_AUTH_LIST={
            "/login",
            "/register",
            "/static/**",
            "/img/**",
            "/css/**",
            "/js/**"
    };

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.csrf().disable().formLogin().loginPage("/login").loginProcessingUrl("/login").usernameParameter(
                        "username").permitAll()
                //失败处理
                .failureHandler((req, resp, e) -> ResponseUtil.restResponse(resp, HttpStatus.FORBIDDEN, RestResult.error(403, e.getMessage())))
                //成功处理
                .successHandler((req ,resp, e)-> ResponseUtil.restResponse(resp, HttpStatus.OK,
                        RestResult.success(200, "登录成功!"))
                .permitAll()
                .and().exceptionHandling()
                //请求登录处理，改变默认跳转登录页
                .authenticationEntryPoint((req, resp, e) -> resp.sendRedirect("/login"))
                //没有权限访问
                .accessDeniedHandler((req, resp, e) -> ResponseUtil.restResponse(resp, HttpStatus.FORBIDDEN, RestResult.error(403, "抱歉，你当前的身份无权访问")))
                //设置最大一人同时登陆
                .and().sessionManagement().maximumSessions(1)
                .expiredSessionStrategy(s -> ResponseUtil.restResponse(s.getResponse(), HttpStatus.FORBIDDEN, RestResult.error(499, "您的账号在别的地方登录，当前登录已失效")))
                .and()
                //设置登出
                .and().logout().logoutUrl("/logout").logoutSuccessUrl("/login").deleteCookies("JSESSIONID").permitAll()
                .and().authorizeRequests().antMatchers(NO_AUTH_LIST).permitAll()
                .antMatchers("/*.svg","/*.png","/*.js","/*.css").permitAll()
                .and().authorizeRequests().anyRequest().authenticated();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

#### 3.2 自定义Detail接口

```java
/**
 * @Describe: 类描述
 * @Author: tyf
 * @CreateTime: 2021/11/15
 **/
public class TeacherDetail extends Teacher implements UserDetails{

    public TeacherDetail(Teacher teacher){
        BeanUtils.copyProperties(teacher, this);
    }
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return null;
    }

    @Override
    public String getPassword() {
        return super.getPassword();
    }

    @Override
    public String getUsername() {
        return super.getName();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}

```

#### 3.3 实现UserDetailService接口

```java
/**
 * @Describe: 类描述
 * @Author: tyf
 * @CreateTime: 2021/11/15
 **/
@Service
public class TeacherServiceImpl implements TeacherService, UserDetailsService {

    @Resource
    private TeacherMapper teacherMapper;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        Teacher teacher = new Teacher();
        teacher.setName(s);
        Example<Teacher> teacherExample = Example.of(teacher);
        Optional<Teacher> resultTeacher = teacherMapper.findOne(teacherExample);
        if(resultTeacher.isPresent()){
            return new TeacherDetail(resultTeacher.get());
        }
        return null;
    }
}

```

### 4.（可选）其他配置

```java
@Component
public class WebMvcConfigurerAdapter implements WebMvcConfigurer {


    @Resource(name="thymeleafViewResolver")
    private ThymeleafViewResolver thymeleafViewResolver;

    @Value("${spring.imageProxy}")
    private String imageProxy = "";


    //Thymeleaf增加全局变量。用来图像加速等；
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {

        if (thymeleafViewResolver != null) {
            Map<String, Object> vars = new HashMap<>(1);
            vars.put("imageProxy", imageProxy);
            thymeleafViewResolver.setStaticVariables(vars);
        }
        WebMvcConfigurer.super.configureViewResolvers(registry);
    }
}
```



## 末、开始编写

1. Mapper
2. Service
3. Controller

## 附：

### 目录结构

```asp
│  StudentMannagerSystemApplication.java
│
├─conf
│      SecurityConfigurer.java
│      WebMvcConfigurerAdapter.java
│
├─controller
│      ScoreController.java
│      StudentController.java
│      TeacherController.java
│
├─enums
│      ResultEnum.java
│
├─mapper
│      ScoreMapper.java
│      StudentMapper.java
│      TeacherMapper.java
│
├─model
│      RestResult.java
│      TeacherDetail.java
│
├─pojo
│      Score.java
│      Student.java
│      Teacher.java
│
├─service
│  │  ScoreService.java
│  │  StudentService.java
│  │  TeacherService.java
│  │
│  └─impl
│          ScoreServiceImpl.java
│          StudentServiceImpl.java
│          TeacherServiceImpl.java
│
└─tools
        PasswordTools.java
        ResponseUtil.java
```



