---
layout:     post
title:      Spring Security
subtitle:   Spring Security学习指南
date:       2021-11-17
author:     Tanyongfeng
header-img: img/69530.jpg
catalog: true
tags:
    - Spring Security
    - Java
    - 后端
---

# Spring Security

![image-20210318210816230](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/image-20210318210816230.png)

## 1. 核心组件

### 1. SecurityContextHolder

SecurityContextHolder它持有的是安全上下文（Security Context）的信息。当前操作的用户是谁，该用户是否已经被认证，他拥有哪些角色权等等，这些都被保存在SecurityContextHolder中。SecurityContextHolder默认使用ThreadLocal 策略来存储认证信息。看到ThreadLocal 也就意味着，这是一种与线程绑定的策略。在web环境下，Spring Security在用户登录时自动绑定认证信息到当前线程，在用户退出时，自动清除当前线程的认证信息。

```java
Object principal =SecurityContextHolder
    	.getContext()		 //获取上下文信息
        .getAuthentication() //获取认证信息
        .getPrincipal(); 	 //获取身份信息
//UserDetails是Sring对身份信息的一个封装的一个接口
if (principal instanceof UserDetails) { 
	String username = ((UserDetails)principal)
        .getUsername();
} else {
    //获取用户名
    String username = princidepal.toString();
}
```

### 2. SecurityContext

安全上下文，主要持有Authentication对象，如果用户未鉴权，那Authentication对象将会是空的。

### 3. Authentication

鉴权对象，该对象主要包含了用户的详细信息（UserDetails）和用户鉴权时所需要的信息，如用户提交的用户名密码、Remember-me Token，或者digest hash值等，按不同鉴权方式使用不同的Authentication实现。

1. Authentication是spring security包中的接口，直接继承自Principal类，而Principal是位于java.security包中的。可以见得，Authentication在spring security中是最高级别的身份/认证的抽象。
    2、 由这个顶级接口，我们可以得到用户拥有的权限信息列表，密码，用户细节信息，用户身份信息，认证信息。
    3、getAuthorities()，权限信息列表，默认是GrantedAuthority接口的一些实现类，通常是代表权限信息的一系列字符串。
    4、getCredentials()，密码信息，用户输入的密码字符串，在认证过后通常会被移除，用于保障安全。
    5、getDetails()，细节信息，web应用中的实现接口通常为 WebAuthenticationDetails，它记录了访问者的ip地址和sessionId的值。
    6、getPrincipal()，敲黑板！！！最重要的身份信息，大部分情况下返回的是UserDetails接口的实现类，也是框架中的常用接口之一。

### 4. UserDetails

这个接口规范了用户详细信息所拥有的字段，譬如用户名、密码、账号是否过期、是否锁定等。在Spring Security中，获取当前登录的用户的信息,一般情况是需要在这个接口上面进行扩展，用来对接自己系统的用户.

```java
public interface UserDetailsService {
	UserDetails loadUserByUsername(String username) throws 		UsernameNotFoundException;
}
```

### 5. UserDetailsService

这个接口只提供一个接口loadUserByUsername(String username)，这个接口非常重要，一般情况我们都是通过扩展这个接口来显示获取我们的用户信息，用户登陆时传递的用户名和密码也是通过这里这查找出来的用户名和密码进行校验，但是真正的校验不在这里，而是由AuthenticationManager以及AuthenticationProvider负责的，需要强调的是，如果用户不存在，不应返回NULL，而要抛出异常UsernameNotFoundException. 

### 6. GrantedAuthority

该接口表示了当前用户所拥有的权限（或者角色）信息。这些信息由授权负责对象AccessDecisionManager来使用，并决定最终用户是否可以访问某资源（URL或方法调用或域对象）。鉴权时并不会使用到该对象。

![preview](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/bVbzNF1)

---

## 2. Spring Security是如何完成身份认证的？

1、用户名和密码被过滤器获取到，封装成Authentication,通常情况下是UsernamePasswordAuthenticationToken这个实现类。

2、AuthenticationManager 身份管理器负责验证这个Authentication

3、认证成功后，AuthenticationManager身份管理器返回一个被填充满了信息的（包括上面提到的权限信息，身份信息，细节信息，但密码通常会被移除）Authentication实例。

4、SecurityContextHolder安全上下文容器将第3步填充了信息的Authentication，通过SecurityContextHolder.getContext().setAuthentication()方法，设置到其中。

**单点登录（Single Sign On）**，简称为SSO，是比较流行的企业业务整合的解决方案之一。 SSO的定义是在多个应用系统中，用户只需要登录一次就可以访问所有相互信任的应用系统。

#### Token

Token是服务端端生成的一串字符串，作为客户端进行请求时辨别客户身份的的一个令牌。当用户第一次登录后，服务器生成一个Token便将此Token返回给客户端，以后客户端只需带上这个Token前来请求数据即可，无需再次带上用户名和密码。

**目的**:

Token的目的是为了验证用户登录情况以及减轻服务器的压力，减少频繁的查询数据库，使服务器更加健壮

**使用基于 Token 的身份验证方法，在服务端不需要存储用户的登录记录。大概的流程是这样的：**

1. 客户端使用用户名跟密码请求登录

2. 服务端收到请求，去验证用户名与密码

3. 验证成功后，服务端会签发一个 Token，再把这个 Token 发送给客户端

4. 客户端收到 Token 以后可以把它存储起来，比如放在 Cookie 里或者 Local Storage 里

5. 客户端每次向服务端请求资源的时候需要带着服务端签发的 Token

6. 服务端收到请求，然后去验证客户端请求里面带着的 Token:

   1. 如果两个 token 值相同， 说明用户登录成功过!当前用户处于登录状态；
   2. 如果没有这个 token 值, 没有登录成功；

   3. 如果 token 值不同: 说明原来的登录信息已经失效,让用户重新登录；

## 源码阅读

[SpringSecurity认证流程](https://wanglinyong.github.io/2018/07/10/Spring-Security%E7%99%BB%E5%BD%95%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E5%8E%9F%E7%90%86/)

### 用户验证登录

1. 首先我们知道Spring Security底层默认使用 "/login"来进行处理登录验证请求, 那么`UsernamePasswordAuthenticationFilter`就是这个功能的过滤器

   ```java
   //用户密码验证器的过滤链
   public class UsernamePasswordAuthenticationFilter extends
   		AbstractAuthenticationProcessingFilter {
       //定义获取用户密码的属性名,这就是为什么我们需要在表单的属性
   	public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
   	public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";
   
   	private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;
   	private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;
   	private boolean postOnly = true;
   
   	//表单必须是 /login post请求
   	public UsernamePasswordAuthenticationFilter() {
   		super(new AntPathRequestMatcher("/login", "POST"));
   	}
   
   
       //返回授权对象
   	public Authentication attemptAuthentication(HttpServletRequest request,
   			HttpServletResponse response) throws AuthenticationException {
   		if (postOnly && !request.getMethod().equals("POST")) {
   			throw new AuthenticationServiceException(
   					"Authentication method not supported: " + request.getMethod());
   		}
   
           //提取request请求中的username和password
   		String username = obtainUsername(request);
   		String password = obtainPassword(request);
   
   		if (username == null) {
   			username = "";
   		}
   
   		if (password == null) {
   			password = "";
   		}
   
   		username = username.trim();
   
   		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
   				username, password);
   
   		// Allow subclasses to set the "details" property
   		setDetails(request, authRequest);
   
           //重点
   		return this.getAuthenticationManager().authenticate(authRequest);
   	}
   ```

2. 上面代码第**40**行, 我们进入`UsernamePasswordAuthenticationToken(username, password)`这个方法

   ```java
   //principal: 首长  credentials: 证书
   public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
       //调用父类的AbstractAuthenticationToken的构造方法, 就是给用户授权, 但是我们还未验证, 所以复制null
       super(null);
       this.principal = principal;
       this.credentials = credentials;
       setAuthenticated(false);
   }
   ```

3. **1**中代码的第44行, 我们调用 `setDetails(request, authRequest)`方法,是将当前的请求信息设置到`UsernamePasswordAuthenticationToken`中

   ```java
   	protected void setDetails(HttpServletRequest request,
   			UsernamePasswordAuthenticationToken authRequest) {
   		authRequest.setDetails(authenticationDetailsSource.buildDetails(request));
   	}
   ```

4. 由上面代码的 **46**行, 我们可以发现, 调用` getAuthenticationManager()`方法, 通过这个方法获取到 `AuthenticationManager`这个对象, 然后再调用`authenticate()`方法来来查找支持该token(UsernamePasswordAuthenticationToken)认证方式的provider，然后调用该provider的authenticate方法进行认证. **`AuthenticationManager`来管理`AuthenticationProvider`的接口**

   ```java
   		return this.getAuthenticationManager().authenticate(authRequest);
   ```

5. 我们找到`AuthenticationManager`的一个实现类`ProviderManager`

   ```java
   public class ProviderManager implements AuthenticationManager, MessageSourceAware,{
       
       //上述调用的authenticate()方法
       public Authentication authenticate(Authentication authentication)
                   throws AuthenticationException {
               Class<? extends Authentication> toTest = authentication.getClass();
               AuthenticationException lastException = null;
               Authentication result = null;
               boolean debug = logger.isDebugEnabled();
   			//通过遍历找到支持验证该Token的provider
               for (AuthenticationProvider provider : getProviders()) {
                   //使用supports来检测这个provider是否支持验证这个Authentication
                   //这里我们找到了abstractUserDetailsAuthenticationProvider
                   if (!provider.supports(toTest)) {
                       continue;
                   }
   
                   if (debug) {
                       logger.debug("Authentication attempt using "
                               + provider.getClass().getName());
                   }
   
                   try {
                       //调用provider的authenticate()方法, 我们点进来
                       result = provider.authenticate(authentication);
   
                       if (result != null) {
                           copyDetails(authentication, result);
                           break;IL
                       }
                   }
                   catch (AccountStatusException e) {.....}
   
               if (result == null && parent != null) {
                   // Allow the parent to try.
                   try {
                       result = parent.authenticate(authentication);
                   }
                   catch (ProviderNotFoundException e) {.........}
               }
   
               if (result != null) {
                   if (eraseCredentialsAfterAuthentication
                           && (result instanceof CredentialsContainer)) {
                       // Authentication is complete. Remove credentials and other secret data
                       // from authentication
                       ((CredentialsContainer) result).eraseCredentials();
                   }
   
                   eventPublisher.publishAuthenticationSuccess(result);
                   return result;
               }
   
               // Parent was null, or didn't authenticate (or throw an exception).
   
               if (lastException == null) {
                   lastException = new ProviderNotFoundException(messages.getMessage(
                           "ProviderManager.providerNotFound",
                           new Object[] { toTest.getName() },
                           "No AuthenticationProvider found for {0}"));
               }
   
               prepareException(lastException, authentication);
   
               throw lastException;
           }
   ```

   **第9行**: 这里的getProviders()会有很多提供验证Token的类

   **第25**: 调用实现类`abstractUserDetailsAuthenticationProvider`的方法

6. 分析上一步的`abstractUserDetailsAuthenticationProvider`的`authenticate`方法

   ```java
   public abstract class AbstractUserDetailsAuthenticationProvider implements
   		AuthenticationProvider, InitializingBean, MessageSourceAware {  
       
     
       public Authentication authenticate(Authentication authentication)
                       throws AuthenticationException {
           Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
                               messages.getMessage(
                                   "AbstractUserDetailsAuthenticationProvider.onlySupports",
                                   "Only UsernamePasswordAuthenticationToken is supported"));
   
           // Determine username
           String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
               : authentication.getName();
   
           boolean cacheWasUsed = true;
        	//从缓存中通过username找到userDetails
           UserDetails user = this.userCache.getUserFromCache(username);
   
           if (user == null) {
               cacheWasUsed = false;
               //开始
               try {
            //通过此方法来找到user对象, 这个在下面的DaoAuthenticationProvider有实现, 这是是从数据库来获得的对象
                   user = retrieveUser(username,
                                       (UsernamePasswordAuthenticationToken) authentication);
               }
               catch (UsernameNotFoundException notFound) {... }
   
           try {
               
               //拿到user对象开始验证操作...先验证用户是否被冻结, 用户未启用等
               preAuthenticationChecks.check(user);
               //验证密码的方法, 具体实现在 实现类
               additionalAuthenticationChecks(user,
                                              (UsernamePasswordAuthenticationToken) authentication);
           }
           catch (AuthenticationException exception) {
               if (cacheWasUsed) {
                   // There was a problem, so try again after checking
                   // we're using latest data (i.e. not from the cache)
                   cacheWasUsed = false;
                   user = retrieveUser(username,
                                       (UsernamePasswordAuthenticationToken) authentication);
                   preAuthenticationChecks.check(user);
                   additionalAuthenticationChecks(user,
                                             (UsernamePasswordAuthenticationToken) authentication);
               }
               else {
                   throw exception;
               }
           }
           //验证用户凭证是否过期
           postAuthenticationChecks.check(user);
           if (!cacheWasUsed) {
               this.userCache.putUserInCache(user);
           }
           Object principalToReturn = user;
           if (forcePrincipalAsString) {
               principalToReturn = user.getUsername();
           }
               //返回已经经过验证的Authentication
           return createSuccessAuthentication(principalToReturn, authentication, user);
       }   
           //第24行所调用的接口, 
       protected abstract UserDetails retrieveUser(String username,
   			UsernamePasswordAuthenticationToken authentication)
   			throws AuthenticationException;
           
           
        //用户验证成功,构造一个成功的Authentication
       protected Authentication createSuccessAuthentication(Object principal,
   			Authentication authentication, UserDetails user) {
   		// Ensure we return the original credentials the user supplied,
   		// so subsequent attempts are successful even with encoded passwords.
   		// Also ensure we return the original getDetails(), so that future
   		// authentication events after cache expiry contain the details
   		UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
   				principal, authentication.getCredentials(),
   				authoritiesMapper.mapAuthorities(user.getAuthorities()));
   		result.setDetails(authentication.getDetails());
   
   		return result;
   	}
   }
   ```

7. 分析`DaoAuthenticationProvider`: 其是上面`AbstractUserDetailsAuthenticationProvider`的实现类, 实现了`retrieveUser`方法

   ```java
   public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
       
       //构造器, 可以看出默认使用 PlaintextPasswordEncoder 这个编码器
       public DaoAuthenticationProvider() {
   		setPasswordEncoder(new PlaintextPasswordEncoder());
   	}
       
       protected final UserDetails retrieveUser(String username,
   			UsernamePasswordAuthenticationToken authentication)
   			throws AuthenticationException {
        
   		UserDetails loadedUser;
   		try {
               //这里通过getUserDetailsService()方法来找到UserDetailsService接口的实现类
               //所以这里解释了为什么我们需要自己User类继承 UseDetails接口
   			loadedUser = this.getL().loadUserByUsername(username);
   		}
   		catch (UsernameNotFoundException notFound) {
   			if (authentication.getCredentials() != null) {
   				String presentedPassword = authentication.getCredentials().toString();
   				passwordEncoder.isPasswordValid(userNotFoundEncodedPassword,
   						presentedPassword, null);
   			}
   			throw notFound;
   		}
   		catch (Exception repositoryProblem) {....}
   
   		if (loadedUser == null) {
   			throw new InternalAuthenticationServiceException;
   		}
   		return loadedUser;
   	}
       
       //验证密码实现类
       protected void additionalAuthenticationChecks(UserDetails userDetails,
   			UsernamePasswordAuthenticationToken authentication)
   			throws AuthenticationException {
   		Object salt = null;
   
   		if (this.saltSource != null) {
   			salt = this.saltSource.getSalt(userDetails);
   		}
   
   		if (authentication.getCredentials() == null) {
   			logger.debug("Authentication failed: no credentials provided");
   
   			throw new BadCredentialsException(messages.getMessage(
   					"AbstractUserDetailsAuthenticationProvider.badCredentials",
   					"Bad credentials"));
   		}
   
   		String presentedPassword = authentication.getCredentials().toString();
   
           //这里是验证密码的方法, 我们直接进入PlaintextPasswordEncoder这个类
   		if (!passwordEncoder.isPasswordValid(userDetails.getPassword(),
   				presentedPassword, salt)) {
   			logger.debug("Authentication failed: password does not match stored value");
   
   			throw new BadCredentialsException(messages.getMessage(
   					"AbstractUserDetailsAuthenticationProvider.badCredentials",
   					"Bad credentials"));
   		}
   	}
   
      //获取 UserDetailService的实现类
       protected UserDetailsService getUserDetailsService() {
   		return userDetailsService;
   	}
   ```

8. 密码验证类`PlaintextPasswordEncoder`

   ```java
   public class PlaintextPasswordEncoder extends BasePasswordEncoder {
       
       //是否忽略大小写
   	private boolean ignorePasswordCase = false;
   
       //encpass:数据库中等获取到的正确的密码  rawPass: 用户输入的密码
   	public boolean isPasswordValid(String encPass, String rawPass, Object salt) {
   		String pass1 = encPass + "";
   
   		// Strict delimiters is false because pass2 never persisted anywhere
   		// and we want to avoid unnecessary exceptions as a result (the
   		// authentication will fail as the encodePassword never allows them)
   		String pass2 = mergePasswordAndSalt(rawPass, salt, false);
   
   		if (ignorePasswordCase) {
   			// Note: per String javadoc to get correct results for Locale insensitive, use
   			// English
   			pass1 = pass1.toLowerCase(Locale.ENGLISH);
   			pass2 = pass2.toLowerCase(Locale.ENGLISH);
   		}
           //返回密码是否相同
   		return PasswordEncoderUtils.equals(pass1, pass2);
   	}
   ```

9. 至此, 我们已经完成了密码的验证, 我们回到第**6**步的63行, 

10. 流程图:

    ![未命名文件](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/未命名文件.png)

11. 相关类图的构造

    ![UsernamePasswordAuthenticationToken](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/UsernamePasswordAuthenticationToken.png)

    ![DaoAuthenticationProvider](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/DaoAuthenticationProvider.png)

    ![image-20210318202953304](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/image-20210318202953304.png)

### 自定义登录表单

**改变默认登录界面**

```java
formLogin().loginPage("/login.html").loginProcessingUrl("/login").permitAll()
    // loginPage配置了自定义登录表单, loginProcessingUrl配置了处理的请求地址,这个需要在自定义表单的 action 字段中一致
   
```

**未认证用户自动跳转原理**:

`ExceptionTranslationFilter`使用一个`AuthenticationEntryPoint`在需要的时候来启动表单认证流程，缺省使用的实现是类`LoginUrlAuthenticationEntryPoint`,它的认证表单的请求提交由`UsernamePasswordAuthenticationFilter`负责处理。 

```java
//LoginUrlAuthenticationEntryPoint源码
public class LoginUrlAuthenticationEntryPoint implements AuthenticationEntryPoint,
		InitializingBean {
	private static final Log logger = LogFactory
			.getLog(LoginUrlAuthenticationEntryPoint.class);
	private PortMapper portMapper = new PortMapperImpl();

	private PortResolver portResolver = new PortResolverImpl();

	private String loginFormUrl; 		//跳转页面 可以是相对路径 也可是绝对路径

	private boolean forceHttps = false; //强行使用https协议

	private boolean useForward = false; 

	private final RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();

	/**
	 *
	 * @param loginFormUrl URL where the login page can be found. Should either be
	 * relative to the web-app context path (include a leading {@code /}) or an absolute
	 * URL.
	 */
	public LoginUrlAuthenticationEntryPoint(String loginFormUrl) {
		Assert.notNull(loginFormUrl, "loginFormUrl cannot be null");
		this.loginFormUrl = loginFormUrl;
	}
```
