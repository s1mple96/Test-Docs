# BUG-【任货行】注销流程静止期处理定时任务空指针异常

## 1. BUG描述

注销流程静止期处理定时任务执行时抛出空指针异常，导致定时任务中断。

**影响**：
- ❌ 定时任务无法正常完成
- ❌ 部分注销订单的静止期状态无法自动更新

---

## 2. 复现步骤

### 前置条件
- 测试环境：http://192.168.1.60:8087/hcbTimer-bill
- 存在超过4天静止期的注销订单
- 问题订单ID：`4f70e7e94eda4cf5a01599d66331ffc4`
- 身份证号：`142229198411114612`

### 操作步骤

**Step 1**: 调用定时任务接口

```bash
curl http://192.168.1.60:8087/hcbTimer-bill/test/cancelBussineOverRestingStageHandle
```

**Step 2**: 查询问题订单的数据

```bash
# 查询注销订单
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
SELECT CANCELBUSSINE_ID, IDCODE, STATUS, IS_NEW_ORDER, CREATE_TIME 
FROM hcb_hlj_cancelbussine 
WHERE CANCELBUSSINE_ID='4f70e7e94eda4cf5a01599d66331ffc4'"

# 查询对应的钱包
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
SELECT * FROM HCB_HLJ_TRUCKUSERWALLET 
WHERE ID_CODE='142229198411114612'"

# 查询对应的车辆
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
SELECT TRUCKUSER_ID, CAR_NUM, STATUS, ID_CODE 
FROM hcb_truckuser 
WHERE TRUCKUSER_ID='d1f7f0d6e6e9446da5a023155a80c479'"
```

**数据查询结果**：
```
# 注销订单存在
CANCELBUSSINE_ID: 4f70e7e94eda4cf5a01599d66331ffc4
IDCODE: 142229198411114612
STATUS: 1
IS_NEW_ORDER: 1
CREATE_TIME: 2025-11-17 11:24:11

# 钱包不存在
(Empty set)

# 车辆不存在
(Empty set)
```

**Step 3**: 观察日志

```bash
ssh test
tail -n 100 /home/java-server/hcbTimer-bill/log.txt
```

---

## 3. 预期结果

```
[INFO] - ============= 进入 注销流程静止期处理 start ==========
[INFO] - 查询到待处理订单 5 条
[INFO] - 处理订单 CANCELBUSSINE_ID=xxx
[INFO] - 查询钱包余额成功
[INFO] - 更新订单状态成功
[INFO] - ============= 进入 注销流程静止期处理 end ==========
```

**预期**：
- ✅ 定时任务正常执行完成
- ✅ 所有订单的静止期状态更新成功
- ✅ 日志输出正常

---

## 4. 实际结果

```
[2026-01-24 17:12:51.091] [hcbTimer-bill] [ERROR] - 注销流程静止期处理定时任务失败null
java.lang.NullPointerException
at com.etc.api.task.CancelBussineTimerTask.overRestingStageHandle(CancelBussineTimerTask.java:140)
at com.etc.api.task.CancelBussineTimerTask$$FastClassBySpringCGLIB$$32232d05.invoke(<generated>)
at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:720)
at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:157)
at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:92)
at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:655)
at com.etc.api.task.CancelBussineTimerTask$$EnhancerBySpringCGLIB$$d72430e9.overRestingStageHandle(<generated>)
at com.etc.api.controller.app.test.TestController.cancelBussineOverRestingStageHandle(TestController.java:5291)
at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.lang.reflect.Method.invoke(Method.java:498)
at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:221)
at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:136)
at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:110)
at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:832)
at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:743)
at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:85)
at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:961)
at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:895)
at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:967)
at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:858)
at javax.servlet.http.HttpServlet.service(HttpServlet.java:635)
at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:843)
at javax.servlet.http.HttpServlet.service(HttpServlet.java:742)
at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
at org.apache.shiro.web.servlet.ProxiedFilterChain.doFilter(ProxiedFilterChain.java:61)
at org.apache.shiro.web.servlet.AdviceFilter.executeChain(AdviceFilter.java:108)
at org.apache.shiro.web.servlet.AdviceFilter.doFilterInternal(AdviceFilter.java:137)
at org.apache.shiro.web.servlet.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:125)
at org.apache.shiro.web.servlet.ProxiedFilterChain.doFilter(ProxiedFilterChain.java:66)
at org.apache.shiro.web.servlet.AbstractShiroFilter.executeChain(AbstractShiroFilter.java:449)
at org.apache.shiro.web.servlet.AbstractShiroFilter$1.call(AbstractShiroFilter.java:365)
at org.apache.shiro.subject.support.SubjectCallable.doCall(SubjectCallable.java:90)
at org.apache.shiro.subject.support.SubjectCallable.call(SubjectCallable.java:83)
at org.apache.shiro.subject.support.DelegatingSubject.execute(DelegatingSubject.java:387)
at org.apache.shiro.web.servlet.AbstractShiroFilter.doFilterInternal(AbstractShiroFilter.java:362)
at org.apache.shiro.web.servlet.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:125)
at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:346)
at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:262)
at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
at com.alibaba.druid.support.http.WebStatFilter.doFilter(WebStatFilter.java:123)
at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
at com.etc.api.filter.CharacterEncodingImplFilter.doFilterInternal(CharacterEncodingImplFilter.java:64)
at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199)
at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:478)
at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:140)
at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81)
at org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:650)
at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87)
at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:342)
at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:803)
at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:868)
at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1459)
at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
at java.lang.Thread.run(Thread.java:748)
[ERROR] - ============= 进入 注销流程静止期处理 end ==========
```

**实际**：
- ❌ 定时任务执行中抛出 `NullPointerException`
- ❌ 异常发生在 `CancelBussineTimerTask.java:140` 行
- ❌ 定时任务中断，后续订单未处理

---

**日志路径**: `/home/java-server/hcbTimer-bill/log.txt`
