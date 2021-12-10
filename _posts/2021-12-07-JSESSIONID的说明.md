---
layout:     post
title:      Jsession使用指南
subtitle:   如何优雅的使用Jessionid
date:       2021-12-10
author:     Tanyongfeng
header-img: img/69530.jpg
catalog: true
tags:
    - SpringBoot
    - 后端
    - Java
---
# JSESSIONID的说明



### JSESSIONID作用

因为HTTP协议是无状态的, 所以我们的服务器不知道对方是谁, 所以事实上, 当用户访问服务器的时候, 服务器会为每一个用户开启一个session, 那么浏览器怎么判断这个session到底属于哪个用户呢, 那么Jsessionid就显现出他的优势了, 服务器可以通过它来找到用户session信息, 它可以帮助服务期判断对方的身份(比如说对方是否已经认证过了....), 用代码表示就是:

```
jsessionid ==request.getSession().getId()
```

![image-20210316140845408](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/image-20210316140845408.png)

### 基本流程

1. 第一次访问服务器的时候, 会在响应头看到Set-Cookie的信息(只有在首次访问的时候在响应头会出现该信息), 随后浏览器会将浏览器的cookie并保存.但是因为没有设置时间, 所以关闭浏览器后就会消失.

   ```properties
   Set-Cookie: JSESSIONID=**********;
   ```

   ![image-20210316140218266](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/image-20210316140218266.png)

2. 当再次请求的时候,浏览器会把请求头里面的cookie发送给服务器(每次请求都是这样)





### Session对象

#### Session类图

![image-20210316202105932](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/image-20210316202105932.png)

- HttpSession是我们大家可以直接使用的Session
- StandardSession是标准的HttpSession的实现, 同时它实现了Session接口, 用于Tomcat内部管理

#### Manager类图

![image-20210316202327687](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/image-20210316202327687.png)

- Manager是用来储存的Session的类, 它会使用Map来储存Session类

  ```java
      /**
       * The set of currently active Sessions for this Manager, keyed by
       * session identifier.
       */
      protected Map<String, Session> sessions = new ConcurrentHashMap<>();
  ```

- 其实现类有`StandardManager`和`PersistentManager`类; 两者的区别:

  - 前者在Tomcat执行的时候, 会把Session储存在内存中, Tomcat关闭的时候,(正常关闭), 会把Session写入到硬盘中, 等到Tomcat重启后将Session写入进来
  - 后者直接将Session存入到硬盘中



#### Manager与Context

在Tomcat中，一個Context就是部署到Tomcat中的一個應用（Webapp）。每一個Context都有一個單獨的Manager物件來管理這個應用的會話資訊。

![image-20210316202824305](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/image-20210316202824305.png)

![image-20210316205748479](https://kauizhaotan.oss-cn-shanghai.aliyuncs.com/img/image-20210316205748479.png)

 ### Session的生命周期

这里我们使用Tomcat作为我们的Web服务器, 那么

1. **解析获取`requestedSession`和`requestSessionID`**:

   `request.getSeesion()`获得Session, 主要调用`doGetSession`方法

   ```java
   //Parameter: create: 在没有获取到Session的情况下, 是否创建一个新的Session
   protected Session doGetSession(boolean create) {
    
       // There cannot be a session if no context has been assigned yet
       //如果没有上下文容器, 那么没有session
       Context context = getContext();
       if (context == null) {
           return (null);
       }
    
       // Return the current session if it exists and is valid
       //如果存在session但是失效, 返回现有的session = null
       if ((session != null) && !session.isValid()) {
           session = null;
       }
       if (session != null) {
           return (session);
       }
    
       // Return the requested session if it exists and is valid
       //如果session可用并且存在
       Manager manager = context.getManager();
       if (manager == null) {
           return null;        // Sessions are not supported
       }
       if (requestedSessionId != null) {
           try {
               //通过我们所说的sessionId来寻找session
               session = manager.findSession(requestedSessionId);
           } catch (IOException e) {
               session = null;
           }
           if ((session != null) && !session.isValid()) {
               session = null;
           }
           if (session != null) {
               session.access();
               return (session);
           }
       }
    
       // Create a new session if requested and the response is not committed
       //否则创建一个Seesion
       if (!create) {
           return (null);
       }
       if ((response != null) &&
               context.getServletContext().getEffectiveSessionTrackingModes().
               contains(SessionTrackingMode.COOKIE) &&
               response.getResponse().isCommitted()) {
           throw new IllegalStateException
           (sm.getString("coyoteRequest.sessionCreateCommitted"));
       }
    
       // Re-use session IDs provided by the client in very limited
       // circumstances.
       //重用session IDs 被客户端提供的
       String sessionId = getRequestedSessionId();
       if (requestedSessionSSL) {
           // If the session ID has been obtained from the SSL handshake then
           // use it.
       } else if (("/".equals(context.getSessionCookiePath())
               && isRequestedSessionIdFromCookie())) {
           /* This is the common(ish) use case: using the same session ID with
            * multiple web applications on the same host. Typically this is
            * used by Portlet implementations. It only works if sessions are
            * tracked via cookies. The cookie must have a path of "/" else it
            * won't be provided for requests to all web applications.
            *
            * Any session ID provided by the client should be for a session
            * that already exists somewhere on the host. Check if the context
            * is configured for this to be confirmed.
            */
           if (context.getValidateClientProvidedNewSessionId()) {
               boolean found = false;
               for (Container container : getHost().findChildren()) {
                   Manager m = ((Context) container).getManager();
                   if (m != null) {
                       try {
                           if (m.findSession(sessionId) != null) {
                               found = true;
                               break;
                           }
                       } catch (IOException e) {
                           // Ignore. Problems with this manager will be
                           // handled elsewhere.
                       }
                   }
               }
               if (!found) {
                   sessionId = null;
               }
           }
       } else {
           sessionId = null;
       }
       //通过manager创建Seesion, 后面会讲到
       session = manager.createSession(sessionId);
    
       // Creating a new session cookie based on that session
       if ((session != null) && (getContext() != null)
               && getContext().getServletContext().
               getEffectiveSessionTrackingModes().contains(
                       SessionTrackingMode.COOKIE)) {
           Cookie cookie =
                   ApplicationSessionCookieConfig.createSessionCookie(
                           context, session.getIdInternal(), isSecure());
           //返回一个带有 SessionID的Cookie
           response.addSessionCookieInternal(cookie);
       }
    
       if (session == null) {
           return null;
       }
    
       session.access();
       return session;
   }
   ```
   
   关于`requestedSessionId`是如何获取的, Tomcat内部可以支持从 Cookie和URL中获取, 具体查看`CoyoteAdapter`类中的 `postPraseRequest:` 

   ```java
   String sessionID;
   if (request.getServletContext().getEffectiveSessionTrackingModes()
           .contains(SessionTrackingMode.URL)) {
    
       //如果浏览器的Cookie被禁用
       //首先尝试在URL中获取, 这里getSessionUriParamName = jsessionid
       sessionID = request.getPathParameter(
               SessionConfig.getSessionUriParamName(
                       request.getContext()));
       if (sessionID != null) {
           request.setRequestedSessionId(sessionID);
           request.setRequestedSessionURL(true);
       }
   }
    
   //在sessionID中获取
   parseSessionCookiesId(req, request);
   ```
   
   具体的`parseSessionCookiesId(req, request)`方法:

   ```java
   //SessionCookieName = JSESSIONID
   String sessionCookieName = SessionConfig.getSessionCookieName(context);
    
   //遍历Cookie, 取出属性名为 JSESSIONID 的字段,
   for (int i = 0; i < count; i++) {
       ServerCookie scookie = serverCookies.getCookie(i);
       if (scookie.getName().equals(sessionCookieName)) {
           // Override anything requested in the URL
           if (!request.isRequestedSessionIdFromCookie()) {
               // Accept only the first session id cookie
               convertMB(scookie.getValue());
               //将读出来的 JSESSIONID 设置在 request上..
               request.setRequestedSessionId
                   (scookie.getValue().toString());
               request.setRequestedSessionCookie(true);
               request.setRequestedSessionURL(false);
               if (log.isDebugEnabled()) {
                   log.debug(" Requested cookie session id is " +
                       request.getRequestedSessionId());
               }
           } else {
               if (!request.isRequestedSessionIdValid()) {
                   // Replace the session id until one is valid
                   convertMB(scookie.getValue());
                   request.setRequestedSessionId
                       (scookie.getValue().toString());
               }
           }
       }
   }
   ```
   
2. **通过findSession()查询Session**

   获取到 `SessionId`后, 回去寻找session, 不同的管理器获取方式不一样, 默认的是`StandardManager`:

   ```java
   protected Map<String, Session> sessions = new ConcurrentHashMap<String, Session>();
    
   public Session findSession(String id) throws IOException {
       if (id == null) {
           return null;
       }
       return sessions.get(id);
   }
   ```

3. **通过creatSession()创建Session**

   ```java
   public Session createSession(String sessionId) {
        
       //拒绝会话
       if ((maxActiveSessions >= 0) &&
               (getActiveSessions() >= maxActiveSessions)) {
           rejectedSessions++;
           throw new TooManyActiveSessionsException(
                   sm.getString("managerBase.createSession.ise"),
                   maxActiveSessions);
       }
        
       // Recycle or create a Session instance
       Session session = createEmptySession();
    
       // Initialize the properties of the new session and return it
       session.setNew(true);
       session.setValid(true);
       session.setCreationTime(System.currentTimeMillis());
       session.setMaxInactiveInterval(((Context) getContainer()).getSessionTimeout() * 60);
       String id = sessionId;
       //如果没有 SessionId 那么会生成一个唯一的 SessionId
       if (id == null) {
           id = generateSessionId();
       }
       //这里我们没有把Session放入到ConcurrentHashMap中, 其实已经在下面这条语句执行过了,可看下一段代码分析
       session.setId(id);
       sessionCounter++;
    
       SessionTiming timing = new SessionTiming(session.getCreationTime(), 0);
       synchronized (sessionCreationTiming) {
           sessionCreationTiming.add(timing);
           sessionCreationTiming.poll();
       }
       return (session);
    
   }
   ```

   ```java
   //SessionID设置完成, 自动放入到 Manager
   public void setId(String id, boolean notify) {
    
       if ((this.id != null) && (manager != null))
           manager.remove(this);
    
       this.id = id;
    
       if (manager != null)
           manager.add(this);
    
       if (notify) {
           tellNew();
       }
   }
   ```

4. **销毁 Session ** 

   Tomcat会定期检测出不活跃的session，然后将其删除，一方面session占用内存，另一方面是安全性的考虑；启动tomcat的同时会启动一个后台线程用来检测过期的session，具体可以查看`ContainerBase`的内部类`ContainerBackgroundProcessor`：

   ```java
   protected class ContainerBackgroundProcessor implements Runnable {
    
        @Override
        public void run() {
            Throwable t = null;
            String unexpectedDeathMessage = sm.getString(
                    "containerBase.backgroundProcess.unexpectedThreadDeath",
                    Thread.currentThread().getName());
            try {
                while (!threadDone) {
                    try {
                        //backgroundProcessorDelay默认值为10, 也就是10秒检测一次
                        Thread.sleep(backgroundProcessorDelay * 1000L);
                    } catch (InterruptedException e) {
                        // Ignore
                    }
                    if (!threadDone) {
                        Container parent = (Container) getMappingObject();
                        ClassLoader cl =
                            Thread.currentThread().getContextClassLoader();
                        if (parent.getLoader() != null) {
                            cl = parent.getLoader().getClassLoader();
                        }
                        //调用processChildren方法
                        processChildren(parent, cl);
                    }
                }
            } catch (RuntimeException e) {
                t = e;
                throw e;
            } catch (Error e) {
                t = e;
                throw e;
            } finally {
                if (!threadDone) {
                    log.error(unexpectedDeathMessage, t);
                }
            }
        }
    
        protected void processChildren(Container container, ClassLoader cl) {
            try {
                if (container.getLoader() != null) {
                    Thread.currentThread().setContextClassLoader
                        (container.getLoader().getClassLoader());
                }
                //调用backgroundProcess();方法, 看下段代码分析
                container.backgroundProcess();
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error("Exception invoking periodic operation: ", t);
            } finally {
                Thread.currentThread().setContextClassLoader(cl);
            }
            Container[] children = container.findChildren();
            for (int i = 0; i < children.length; i++) {
                if (children[i].getBackgroundProcessorDelay() <= 0) {
                    processChildren(children[i], cl);
                }
            }
     }
    }
   ```
   
   ```java
   //Manager的方法
   //接上47行代码 Container类的backgroundProcess() 方法
   public void backgroundProcess() {
       //
       count = (count + 1) % processExpiresFrequency;
       if (count == 0) //如果達到檢查頻率則開始檢查
           processExpires();
   }
    
   /**
    * Invalidate all sessions that have expired.
    */
   public void processExpires() {
    
       long timeNow = System.currentTimeMillis();
       Session sessions[] = findSessions();//獲取所有session物件
       int expireHere = 0 ;//过期session的数量，不要被這個變數名騙了
        
       if(log.isDebugEnabled())
           log.debug("Start expire sessions " + getName() + " at " + timeNow + " sessioncount " + sessions.length);
       //逐个检测Session查看是否过期
       for (int i = 0; i < sessions.length; i++) {
           //如何检测session是否过期呢?
           if (sessions[i]!=null && !sessions[i].isValid()) {
               expireHere++;
           }
       }
       long timeEnd = System.currentTimeMillis();
       if(log.isDebugEnabled())
            log.debug("End expire sessions " + getName() + " processingTime " + (timeEnd - timeNow) + " expired sessions: " + expireHere);
       processingTime += ( timeEnd - timeNow );
    
   }
   ```
   
   ```java
   /**
    * Return the <code>isValid</code> flag for this session.
    */
   @Override
   //判断Session是否过期
   public boolean isValid() {
   
       if (!this.isValid) {
           return false;
       }
   
       if (this.expiring) {
           return true;
       }
   
       if (ACTIVITY_CHECK && accessCount.get() > 0) {
           return true;
       }
   
       //關鍵所在
       //如果有設定最大空閒時間
       //就獲取此Session的空閒時間進行判斷
       //如果已超時，則執行expire操作
       if (maxInactiveInterval > 0) { 
           int timeIdle = (int) (getIdleTimeInternal() / 1000L);
           if (timeIdle >= maxInactiveInterval) {
               expire(true);
           }
       }
   
       return this.isValid;
   }
   ```



### Spring Session

**Spring Session官方解释**

>Spring Session provides an API and implementations for managing a user’s session information. It also provides transparent integration with:
>
>- HttpSession
>
>  \- allows replacing the HttpSession in an application container (i.e. Tomcat) neutral way. Additional features include:
>
>  - **Clustered Sessions**() - Spring Session makes it trivial to support [clustered sessions](https://docs.spring.io/spring-session/docs/1.3.1.RELEASE/reference/html5/#httpsession-redis) without being tied to an application container specific solution.
>  - **Multiple Browser Sessions** - Spring Session supports [managing multiple users’ sessions](https://docs.spring.io/spring-session/docs/1.3.1.RELEASE/reference/html5/#httpsession-multi) in a single browser instance (i.e. multiple authenticated accounts similar to Google).
>  - **RESTful APIs** - Spring Session allows providing session ids in headers to work with [RESTful APIs](https://docs.spring.io/spring-session/docs/1.3.1.RELEASE/reference/html5/#httpsession-rest)
>
>- [WebSocket](https://docs.spring.io/spring-session/docs/1.3.1.RELEASE/reference/html5/#websocket) - provides the ability to keep the `HttpSession` alive when receiving WebSocket messages



其具体的特性非常之多，具体的内容可以从文档中了解到，笔者做一点自己的总结，Spring Session的特性包括但不限于以下：

- 使用GemFire来构建C/S架构的HttpSession（不关注）
- 使用第三方仓储来实现集群session管理，也就是常说的分布式session容器，替换应用容器（如tomcat的session容器）。仓储的实现，Spring Session提供了三个实现（Redis，mongodb，jdbc），其中Redis使我们最常用的。程序的实现，使用AOP技术，几乎可以做到透明化地替换。（核心）
- 可以非常方便的扩展Cookie和自定义Session相关的Listener，Filter。
- 可以很方便的与Spring Security集成，增加诸如findSessionsByUserName，rememberMe，限制同一个账号可以同时在线的Session数（如设置成1，即可达到把前一次登录顶掉的效果）等等

