---
layout:     post
title:      SpringBoot入门
subtitle:   SpringBoot 入门
date:       2021-12-10
author:     Tanyongfeng
header-img: img/69530.jpg
catalog: true
tags:
    - SpringBoot
    - 后端
    - Java
---

# SpringBoot入门

## 初始

### 1. 导入的依赖

```xml
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.0.RELEASE</version>
 </parent>    
```

```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

 starter是场景启动器

SpringBoot将所有的功能场景都抽取出来,做成一个个启动器,引入已依赖就会导入场景,

## 2.主程序类

@SprinBootApplication:SpringBoot应用说明这是个主配置类

## 3.目录

1. resource目录
   - static:保存静态资源 js css images
   - templates:保存所有模板页面(默认不支持Jsp页面) 可以使用模板引擎(themleaf freeMarker)
   - application-properties :Spring Boot 默认配置文件可以修改默认配置

## (一)配置文件

1. 配置文件

   - application.properties文件

   - application.yml文件

   **作用**: 修改自动配置的默认值

    YAML是一个标记语言,以数据为中心, 适合做配置文件

   标记语言:大多使用的是xxx.xml文件;大小写敏感

2. 值的写法

   K:V:字面直接来写

   双引号:不会转

3. 对象 map

   ```yaml
   friends
   	lastname: zhangsan
   	age: 3
   friends: {lastname: zhangsan,age: 18}
   ```

4. 数组

   ```yaml
   pets:
    - cat
    - dog
    - pig
   pets 
   ```

### 1. 配置文件属性注入

|                          | @value       | @ConfigurationProperties |
| ------------------------ | ------------ | :----------------------- |
| 功能                     | 一个一个绑定 | 按照配置文件批量批量绑定 |
| 松散绑定(大小写是否区分) | 不支持       | 支持                     |
| SpEL ( #{ } )            | 支持         | 不支持                   |
| JSR303数据校验(注入时)   | 不支持       | 支持                     |
| 复杂类型封装             | 不支持       | 支持                     |

**使用场景**:

1. 如果说,我们只是在某个业务逻辑中获取一下配置文件中的值,那么就用@Value
2. 如果说我们专门编写一个JavaBean和配置文件进行映射.我们就直接用@Configurtion来注解

### 2. 配置文件导入

1. @ConfigurationProperties(value = ""): value是配置文件路径

2. @ImportResource: 标注在配置类中, 导入配置文件 让配置文件里面内容生效

3. springboot推荐使用**全注解**的方式: 使用@Bean来给Bean添加注解

```java
@Configuration //这是一个配置类
public class MyConfig{
 
    @Bean
    public HelloService helloService(){
        return new HelloService ;
    }
}
```



### 3. 配置文件占位符

可以在配置文件中添加占位符号

```properties
#随机数
${random.int} ${random.long}...
#属性配置
${想获取的值}
#例子
person.lastname=张三${random.int}}
person.list=a,b,c
person.map.k1=${person.lastname}
person.map.k2=v2
```

### 4.多Profile文件

1. 多Profile文件 文件名可以是 xxx-dev.xml / yml然后再指定哪个配置文件使用

2. YAML文件激活 多文档块方式

   ```yaml
   server:
     port: 8081
   spring:
     profiles:
       active: prod   #激活prod环境
   ---
   
   server:
     port: 8083
   spring:
     profiles:dev
   ---
   server:
     port: 8085
   spring:
     profiles:
   ```

3. 激活指定命令行参数

   1. 主配置文件中指定 spring.profiles.active=dev

   2. 命令行:

      java -jar 名字 --spring.profiles.active=dev

   3. 虚拟机参数

      -Dspring.profiles.active=dev

### 5.配置文件加载位置

1. 优先级 项目启动会加载application.yml 或者是 application.xml

   ```
   file:./config/  优先加载file根目录的config目录
   file:/ 
   classpath:/config/
   classpath:/
   全部加载只不过会覆盖
   优先级从高到低所有文件都会被加载 高优先级配置内容会覆盖低优先级的内容
   形成互补配置:
   ```

   还可以通过==spirng.config.location==来改变配置文件位置

   可以使用命令行参数的形式,启动项目的时候来指定配置文件的新位置.
   
   ![image-20210119104138476](http://imtyf.icu/typora_img/image-20210119104138476.png)

### 6.配置文件自动处理(原理)

```java
@Configuration(proxyBeanMethods = false) //表示是一个配置类，和以前的配置文件不一样，可以给容器添加组件
@EnableConfigurationProperties(ServerProperties.class) //将配置文件引入进来
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)//判断是否满足条件，满足则整个配置类的所有配置就会生效
@ConditionalOnClass(CharacterEncodingFilter.class)//判断有无这个类
@ConditionalOnProperty(prefix = "server.servlet.encoding", value = "enabled", matchIfMissing = true)//判断是否存在这个配置，如果在配置中不配置也会默认生效
public class HttpEncodingAutoConfiguration {
    //已经和配置文件中产生映射了
    private final Encoding properties;
    
    @Bean//给容器中添加一个组件 这个组件的某些值会从properties属性中过去
	@ConditionalOnMissingBean
	public CharacterEncodingFilter characterEncodingFilter() {
		CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
		filter.setEncoding(this.properties.getCharset().name());
		filter.setForceRequestEncoding(this.properties.shouldForce(Encoding.Type.REQUEST));
		filter.setForceResponseEncoding(this.properties.shouldForce(Encoding.Type.RESPONSE));
		return filter;
	}
```

精髓：

1. SpringBoot启动会加载大量的自动配置类
2. 我们需要看需要的功能有没有SpringBoot默认写好的自动配合配置类

### 7.细节

1. @Condition注解派生类讲解

   | 注解                       | 作用                                   |
   | -------------------------- | -------------------------------------- |
   | @ConditionOnJava           | 系统的Java版本是否符合指定版本         |
   | @ConditionOnClass          | 系统中是否有指定的类                   |
   | @ConditionOnProperty       | 系统中指定的属性是否有指定的值         |
   | @ConditionOnResource       | 系统中类路径下是是否有指定的资源问夹你 |
   | @ConditionOnWebApplication | 系统是否为Web环境                      |

    **自动配置类只有在一定的条件下才能生效；**

   我们怎么才能知道哪一个类生效

   ```java
   首先在properties中配置debug=true；
   启动项目
   控制台打印出
   -----------------
   Positive matches:匹配到的项目
   -----------------
   
      AopAutoConfiguration matched:
         - @ConditionalOnProperty (spring.aop.auto=true) matched (OnPropertyCondition)
   -------------------
   Negative matches:没有匹配到的项目
   -----------------
   
      ActiveMQAutoConfiguration:
         Did not match:
            - @ConditionalOnClass did not find required class 'javax.jms.ConnectionFactory' (OnClassCondition)
   ```


# (二)日志

### 1.日志框架

常见框架：

JUL、JCL、log4j、slf4j

| 日志门面                           | 日志实现                       |
| ---------------------------------- | ------------------------------ |
| ~~JCL~~、SLF4J、 ~~Jboss-logging~~ | Log4j 、~~JUL~~、Log4j Logback |

左边选一个门面、右边一个实现

日志门面：slf4j

### 2.SLF4J的使用

### 	1.如何系统调用slf4j

开发的时候调用抽象层，不应该直接调用实现层

### 	2. 遗留问题

**如何将系统中其他的日志统一导入到slf4j**

1. 将系统中其他日志框架先排除出去
2. 用中间包来替换原有的日志框架
3. 导入slf4j其他的实现

### 3.SpringBoot日志使用

1. 总结：SpringBoot底层也是使用slf4J+logback的方式进行日志记录
2. SpringBoot也把其他日志替换成了log4j

### 4.日志使用

```java
 Logger logger = LoggerFactory.getLogger(getClass());
    @Test
    public void contextLoad(){
        //日志级别：有底
        logger.trace("这是跟踪轨迹日志");
        logger.debug("这是debug日志");
        //spring默认使用的是info级别root级别
        logger.info("这是info日志");
        logger.warn("这是warn日志");
        logger.error("这是error日志");
    }
```

默认当前项目下生成日志文件。

### 5.指定配置

logback.xml:直接被日志框架识别

logback-spring.xml:日志框架就不直接加载日志的配置项，有SpringBoot框架解析日志配置，可以使用SpringBoot的高级Profile功能

# (三)SpringBoot与Web开发

使用SpringBoot项目

访问路径：

1. 访问warjars/**

```xml
<!--        引入依赖-->
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>jquery</artifactId>
            <version>3.3.1</version>
        </dependency>
```

访问 localhost:8080/webjars/jquery/3.3.1/jquery.js

2. 访问“/**"访问当前项目的任何资源（静态资源的文件夹）

   ```
   
   ```

3. 欢迎页：静态资源下的所有index.html页面 被”/**“映射

   localhost：

## 模板引擎

springBoot推荐使用thymeleaf

热更新按钮 ==Ctrl + Shift + F9==

语法简单 功能强大

1. 引入thymeleaf

2. thmeleaf使用

   ```java
   public class ThymeleafProperties {
   
   	private static final Charset DEFAULT_ENCODING = StandardCharsets.UTF_8;
   	public static final String DEFAULT_PREFIX = "classpath:/templates/";
   	public static final String DEFAULT_SUFFIX = ".html";
       //只要吧HTML页面放在classpath:/templates/,thymeleaf就能自动渲染
   ```

   1. 导入命名空间

      ```html
      <html xmlns:th="http://www.thymeleaf.org">
      ```

   2. 使用

      ```html
      <!DOCTYPE html>
      <html xmlns:th="http://www.thymeleaf.org">
      <head>
          <meta charset="UTF-8">
          <title>Title</title>
      </head>
      <body>
      <h1> 成功运行thymeleaf模板引擎</h1>
      <div th:text="${1}">这是显示模板引擎</div>
      </body>
      </html>
      ```

   ### 语法规则

   1. 命名标签

   2. 表达式语法

      ```properties
      Simple expressions:
      Variable Expressions: ${...}：获取变量值 //看文档
      Selection Variable Expressions: *{...} ：变量的选择表达式 和上一个${}作用一样
      Message Expressions: #{...}
      Link URL Expressions: @{...}
      Fragment Expressions: ~{...}
      Literals
      Text literals: 'one text' , 'Another one!' ,…
      Number literals: 0 , 34 , 3.0 , 12.3 ,…
      Boolean literals: true , false
      Null literal: null
      Literal tokens: one , sometext , main ,…
      Text operations:
      String concatenation: +
      Literal substitutions: |The name is ${name}|
      Arithmetic operations:
      Binary operators: + , - , * , / , %
      Minus sign (unary operator): -
      Boolean operations:
      Binary operators: and , or
      Boolean negation (unary operator): ! , not
      Comparisons and equality:
      Comparators: > , < , >= , <= ( gt , lt , ge , le )
      Equality operators: == , != ( eq , ne )
      Conditional operators:
      If-then: (if) ? (then)
      If-then-else: (if) ? (then) : (else)
      Default: (value) ?: (defaultvalue)
      ```


## SpringMVC自动配置

1. **org.springframework.boot.autoconfigure.web:web所有自动配置场景:**

2. 全面接管SpringMvc

   SpringBoot对SpringMvc中 配置类增加注解@EnableMvc 那么自动配置类会全面失效 

   原理

   1. @EnableWebMvc的核心

      ```java
      public @interface EnableWebMvc {
      }
      ```

   2. 

      ```java
      @Configuration(proxyBeanMethods = false)
      public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
      ```

   3. ```java
      @Configuration(proxyBeanMethods = false)
      @ConditionalOnWebApplication(type = Type.SERVLET)
      @ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
      //容器中没有这个组件时候,这个自动配置才生效
      @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
      @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
      @AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
      		ValidationAutoConfiguration.class })
      public class WebMvcAutoConfiguration {
      ```

   4. @EnableWebMvc将WebMvcConfigurationSupport组件导进来了,

   5. 导入的WebMvcConfigurationSupport只是SpringMvc最基本的功能

3. 如何修改默认配置

   模式:

   1. SpirngBoot在自动配置很多组件的时候,先看容器中有没有用户自己配置的,如果有就用用户配置的,如果没有,才自动配置.如果有些组件有多个,那么就将用户配置的和默认的组合起来.
   2. 在SpirngBoot中,会有非常多XXXCongfigurer,帮助我们配置扩展的

## CRUD

#### 默认第一个功能

访问首页

### 国际化

1. 编写国际化配文件
2. 使用ResourceBundleMessageSource管理国际化资源文件
3. 在页面使用fmt:message去处国际化内容

步骤:

1. 编写国际化配置文件抽取页面需要显示的国际化信息

2. Springboot已经自动配置👌了国际资源文件的组件

   ```java
   	private String basename = "messages";//默认配置在message目录下
   @Bean
   	public MessageSource messageSource(MessageSourceProperties properties) {
   		ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
   		if (StringUtils.hasText(properties.getBasename())) {
               //设置国际化资源化文件的基础名字  
   			messageSource.setBasenames(StringUtils
   					.commaDelimitedListToStringArray(StringUtils.trimAllWhitespace(properties.getBasename())));
   		}
   		if (properties.getEncoding() != null) {
   			messageSource.setDefaultEncoding(properties.getEncoding().name());
   		}
   		messageSource.setFallbackToSystemLocale(properties.isFallbackToSystemLocale());
   		Duration cacheDuration = properties.getCacheDuration();
   		if (cacheDuration != null) {
   			messageSource.setCacheMillis(cacheDuration.toMillis());
   		}
   		messageSource.setAlwaysUseMessageFormat(properties.isAlwaysUseMessageFormat());
   		messageSource.setUseCodeAsDefaultMessage(properties.isUseCodeAsDefaultMessage());
   		return messageSource;
   	}
   ```

   3. 去页面获取国际化的值
   4. 效果:根据浏览器信息来切换效果

   原理:国际化Locale(区域信息对象,SpringMv)

   ```java
   	@Bean
   		@ConditionalOnMissingBean
   		@ConditionalOnProperty(prefix = "spring.mvc", name = "locale")
   		public LocaleResolver localeResolver() {
   			if (this.mvcProperties.getLocaleResolver() == WebMvcProperties.LocaleResolver.FIXED) {
   				return new FixedLocaleResolver(this.mvcProperties.getLocale());
   			}
   			AcceptHeaderLocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
   			localeResolver.setDefaultLocale(this.mvcProperties.getLocale());
   			return localeResolver;
   		}
   默认的就是根据请求头获取区域信息
   ```


### 登录

模板引擎修改,要实时生效

1. 禁用模板引擎的缓存  如何禁用模板有引擎
2. 页面修改后 按crtl+F9 重新编译 

登录错误的表达式  

```html
<p style="color: red" th:text="${msg}" th:if="${not #strings.isEmpty(msg)}"></p>
```

### 拦截器机制

1. 首先编写拦截器
2. 在配置拦截器

```java
public class LoginHandlerInterceptor  implements HandlerInterceptor {
    //方法执行之前
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        Object user = request.getSession().getAttribute("loginUser");
        if (user == null){
            //未登录返回登录页面
            request.setAttribute("msg","没有登录权限");
            request.getRequestDispatcher("/index.html").forward(request,response);
            return false;
        }else{
            //放行请求
            return  true;
        }
```

```java
    //注册拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //SPringBoot已经做好了静态资源映射了
        registry.addInterceptor(new LoginHandlerInterceptor()).addPathPatterns("/**")
                .excludePathPatterns("/index.html","/","/user/login","/static/**","/webjars/**");
    }
```

### 员工列表功能

实验要求:

1. RestfulCRUD:CRUD满足Rest风格  资源名称/资源标识

   |      | 普通CRUD      | Restful风格 |
   | ---: | ------------- | ----------- |
   | 查询 | getREmp       | GET         |
   | 添加 | addEmp?xxx    | POST        |
   | 修改 | updateEmp?id= | PUT         |
   | 删除 | deleteEmp?id= | DELETE      |

   三种引入公共片段的方法

   th:insert:将公共片段插入到div声明元素中

   th:include:将被引入的内容包进这个标签中​

   th:replace:将公共片段引入的元素替换为公共片段

员工提交格式不对

日期的格式化:SpringMVC将页面提交的值需要转换为指定的类型

涉及到类型转换,默认日期是按照/的方式

###   错误机制

默认效果:

1. 返回一个默认的错误页面
2. 如果是其他客户端 会返回一个Json数据

原理

1. DefaultErrorAttributes:帮我们在页面共享信息
2. BasicErrorController:基本错误处理器处理默认的 /error 请求
3. ErrorPageCustomizer:系统出现错误以后,来到error请求进行处理,定制错误页面规则 一旦发生错误

步骤:一旦系统出现4XX和5XX之类的错误;就会被BasicErrorController处理

如何定制:

1. 定制错误页面

   1. **有模板引擎的情况下: error/状态码 将错误页面命名为  错误状态码.html放在模板引起文件夹里面  我们可以吧 4XX.html作为错误页面**
   2. ,没有模板引擎的状况下,会去静态资源文件夹去找
   3. 如果都没有 以上没有错误也页面  就是来到默认的错误页面

   可以获取的属性质:

   1. timestamp:时间戳
   2. status:状态码
   3. error:错误码
   4. exception:异常对象
   5. message:异常信息

2. 定制错误数据

   1. 自定义返回Json数据

## 注册三大组件

1. Servlet Filter Listener 使用 ServleRegistrationBean.....FilterRegistrationBean.....ListenerRegistrationBean.....
2. 使用其他servle容器 我们也可以使用 Jetty 和 undertow 等servlet容器

### 嵌入式容器自动配置原理

```
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication
@EnableConfigurationProperties(ServerProperties.class)
public class EmbeddedWebServerFactoryCustomizerAutoConfiguration {

	/**
	 * Nested configuration if Tomcat is being used.
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Tomcat.class, UpgradeProtocol.class }) //判断是否有类也来
	public static class TomcatWebServerFactoryCustomizerConfiguration {

		@Bean
		public TomcatWebServerFactoryCustomizer tomcatWebServerFactoryCustomizer(Environment environment,
				ServerProperties serverProperties) {
			return new TomcatWebServerFactoryCustomizer(environment, serverProperties);
		}

```

```java
	@Override
	public WebServer getWebServer(ServletContextInitializer... initializers) {
		if (this.disableMBeanRegistry) {
			Registry.disableRegistry();
		}
        //创建一个tomcat容器
		Tomcat tomcat = new Tomcat();
		File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
		tomcat.setBaseDir(baseDir.getAbsolutePath());
		Connector connector = new Connector(this.protocol);
		connector.setThrowOnFailure(true);
		tomcat.getService().addConnector(connector);
		customizeConnector(connector);
		tomcat.setConnector(connector);
		tomcat.getHost().setAutoDeploy(false);
		configureEngine(tomcat.getEngine());
		for (Connector additionalConnector : this.additionalTomcatConnectors) {
			tomcat.getService().addConnector(additionalConnector);
		}
		prepareContext(tomcat.getHost(), initializers);
        //调用的是tomcat方法 将tomcat服务器启动   启动Tomcat服务器
		return getTomcatWebServer(tomcat);
	}
```

配置修改如何生效：

外置的Servlet服务器  步骤

1. 必须创建一个war项目
2. 将嵌入式的Tomcat指定为provided
3. 编写一个SpringBootServletInitializer子类
4. 启动服务器

### 原理

执行Jar包的时候：执行SpringBoot主类的main方法， 启动IOC容器 创建嵌入式的Servlet容器

执行War包的时候：先启动服务器 ，再启动SpringBoot应用（SpringBootServletInitializer） 启动ioc容器



servlet3.0:

1. 服务器启动 会创建当前web应用里面没一个SpringBootServletInitializer实现

2. ServletContainerInitializer 必须绑定在META-INF/services 目录下

## Docker安装

docker主机:安装了Docker程序的机器

docker客户端:链接Docker主机进行操作

docker仓库:保存打包好的软件镜像

docker镜像:软件打包好的软件应用程序

### 1.Docker使用

1. 检查内核版本`uname -r` 必须是3.10以上

2. 安装docker 

   ```
   [root@iZ8vbidmq7xeac0bjjwnztZ ~]# systemctl start docker
   [root@iZ8vbidmq7xeac0bjjwnztZ ~]# docker -v
   ```

3. 停止docker

   ```
   systemctl stop docker 
   ```




## Spring的数据库访问

```yaml
spring:
  datasource:
    username: rootone
    password: 123456
    url: jdbc:mysql://39.98.121.54:3306/jdbc
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.mysql.cj.jdbc.MysqlConnectionPoolDataSourc
    schema:
    	-classpath:     //在开始时候运行sql文件

```

数据源：class com.zaxxer.hikari.HikariDataSource 数据源



## SpringBoot缓存

```java
@Cacheable()
//将方法得结果进行缓存,直接从缓存种读取
几个属性:
	cahceNames：缓存组件的名字
     key：指定缓存数据的key
     keyGenerator:key生成器 可以使用指定的key的生层器
     Unless:否定缓存
         
   
```

运行流程：

1. 方法运行之前，先去查询Cache（缓存组件） 然后 

https://spring.io/

https://spring.io/images/spring-logo-9146a4d3298760c2e7e49595184e1975.svg