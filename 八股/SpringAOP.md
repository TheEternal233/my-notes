# SpringAOP

## 什么是AOP

> **面向切面编程**，将那些与业务无关，但对多个对象产生影响的公共行为和逻辑，抽取成公共模块复用，降低耦合

## 项目中有没有用到AOP

记录操作日志，spring实现的事务，缓存

> 核心是：使用aop的环绕通知+切点表达式（找到要记录日志的方法），通过环绕通知的参数获取请求方法的参数（类，方法，注解，请求方式等），获得这些参数后，保存到数据库

## Spring中事务是如何实现的

> 底层是AOP，对方法前后进行拦截，方法开始时开始事务，结束后根据具体情况选择提交或者回滚事务

## SpringAOP的实现实例（以操作日志为例）

自定义操作日志注解

~~~java
package com.example.demo.annotation;

import java.lang.annotation.*;

/**
 * 操作日志注解
 * 标记在方法上，记录操作行为
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface OperationLog {
    
    /** 操作模块 */
    String module() default "";
    
    /** 操作类型：ADD/UPDATE/DELETE/QUERY/EXPORT/LOGIN 等 */
    String type() default "";
    
    /** 操作描述 */
    String desc() default "";
    
    /** 是否保存请求参数 */
    boolean saveParams() default true;
    
    /** 是否保存响应结果 */
    boolean saveResult() default false;
}
~~~

操作日志实体类****

~~~java
package com.example.demo.entity;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@TableName("sys_operation_log")
public class OperationLogEntity {
    
    @TableId(type = IdType.AUTO)
    private Long id;
    
    /** 操作模块 */
    private String module;
    
    /** 操作类型 */
    private String type;
    
    /** 操作描述 */
    private String description;
    
    /** 请求方法 */
    private String requestMethod;
    
    /** 请求URL */
    private String requestUrl;
    
    /** 请求参数 */
    private String requestParams;
    
    /** 响应结果 */
    private String responseResult;
    
    /** 操作人ID */
    private Long operatorId;
    
    /** 操作人名称 */
    private String operatorName;
    
    /** 操作IP */
    private String ip;
    
    /** 操作地点 */
    private String location;
    
    /** 浏览器 */
    private String browser;
    
    /** 操作系统 */
    private String os;
    
    /** 执行耗时(ms) */
    private Long executionTime;
    
    /** 操作状态：0-失败 1-成功 */
    private Integer status;
    
    /** 错误信息 */
    private String errorMsg;
    
    /** 创建时间 */
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
}
~~~

**操作日志切面（核心代码）**

~~~java
package com.example.demo.aspect;

import com.alibaba.fastjson2.JSON;
import com.example.demo.annotation.OperationLog;
import com.example.demo.entity.OperationLogEntity;
import com.example.demo.service.OperationLogService;
import eu.bitwalker.useragentutils.UserAgent;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;
import java.lang.reflect.Method;
import java.time.LocalDateTime;
import java.util.Arrays;
import java.util.stream.Collectors;

@Slf4j
@Aspect
@Component
public class OperationLogAspect {
    
    @Autowired
    private OperationLogService operationLogService;
    
    /** 配置切入点：所有带有 @OperationLog 注解的方法 */
    @Pointcut("@annotation(com.example.demo.annotation.OperationLog)")
    public void operationLogPointcut() {}
    
    /**
     * 环绕通知：记录操作日志核心逻辑
     * 使用 @Around 可以获取方法执行前后的完整信息
     */
    @Around("operationLogPointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        // 1. 获取方法上的注解信息
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        OperationLog annotation = method.getAnnotation(OperationLog.class);
        
        // 2. 获取请求信息
        ServletRequestAttributes attributes = 
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes != null ? attributes.getRequest() : null;
        
        // 3. 构建日志实体
        OperationLogEntity logEntity = new OperationLogEntity();
        logEntity.setModule(annotation.module());
        logEntity.setType(annotation.type());
        logEntity.setDescription(annotation.desc());
        logEntity.setRequestMethod(request != null ? request.getMethod() : "UNKNOWN");
        logEntity.setRequestUrl(request != null ? request.getRequestURI() : "");
        
        // 4. 记录请求参数
        if (annotation.saveParams()) {
            String params = Arrays.stream(joinPoint.getArgs())
                .map(arg -> arg != null ? arg.toString() : "null")
                .collect(Collectors.joining(", "));
            // 敏感信息脱敏处理（如密码）
            logEntity.setRequestParams(desensitize(params));
        }
        
        // 5. 记录操作人信息（从 SecurityContext 或 Session 获取）
        // logEntity.setOperatorId(getCurrentUserId());
        // logEntity.setOperatorName(getCurrentUserName());
        
        // 6. 记录IP和浏览器信息
        if (request != null) {
            logEntity.setIp(getClientIp(request));
            UserAgent userAgent = UserAgent.parseUserAgentString(request.getHeader("User-Agent"));
            logEntity.setBrowser(userAgent.getBrowser().getName());
            logEntity.setOs(userAgent.getOperatingSystem().getName());
        }
        
        // 7. 记录开始时间并执行目标方法
        long startTime = System.currentTimeMillis();
        Object result = null;
        Throwable ex = null;
        
        try {
            // 执行目标方法
            result = joinPoint.proceed();
            logEntity.setStatus(1); // 成功
        } catch (Throwable throwable) {
            ex = throwable;
            logEntity.setStatus(0); // 失败
            logEntity.setErrorMsg(throwable.getMessage());
            throw throwable; // 继续抛出异常，不影响业务
        } finally {
            // 8. 计算耗时
            long executionTime = System.currentTimeMillis() - startTime;
            logEntity.setExecutionTime(executionTime);
            
            // 9. 记录响应结果（可选）
            if (annotation.saveResult() && result != null) {
                logEntity.setResponseResult(JSON.toJSONString(result));
            }
            
            // 10. 异步保存日志
            operationLogService.saveLogAsync(logEntity);
            
            // 控制台输出（调试用）
            log.info("[操作日志] {} - {} - 耗时:{}ms - 状态:{}", 
                annotation.module(), 
                annotation.desc(),
                executionTime,
                logEntity.getStatus() == 1 ? "成功" : "失败");
        }
        
        return result;
    }
    
    /**
     * 敏感信息脱敏
     */
    private String desensitize(String params) {
        if (params == null) return "";
        // 简单示例：替换密码字段
        return params.replaceAll("(\"password\"\\s*:\\s*\")[^\"]*\"", "$1***\"");
    }
    
    /**
     * 获取客户端真实IP
     */
    private String getClientIp(HttpServletRequest request) {
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        // 多个代理情况，取第一个IP
        if (ip != null && ip.contains(",")) {
            ip = ip.split(",")[0].trim();
        }
        return ip;
    }
}
~~~

业务使用示例

~~~java
package com.example.demo.controller;

import com.example.demo.annotation.OperationLog;
import com.example.demo.entity.User;
import com.example.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    /**
     * 查询用户 - 记录操作日志
     */
    @OperationLog(module = "用户管理", type = "QUERY", desc = "查询用户信息", saveParams = true, saveResult = false)
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }
    
    /**
     * 新增用户 - 记录操作日志
     */
    @OperationLog(module = "用户管理", type = "ADD", desc = "新增用户", saveParams = true, saveResult = true)
    @PostMapping
    public User addUser(@RequestBody User user) {
        userService.save(user);
        return user;
    }
    
    /**
     * 修改用户 - 记录操作日志
     */
    @OperationLog(module = "用户管理", type = "UPDATE", desc = "修改用户信息")
    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        user.setId(id);
        userService.updateById(user);
        return user;
    }
    
    /**
     * 删除用户 - 记录操作日志
     */
    @OperationLog(module = "用户管理", type = "DELETE", desc = "删除用户")
    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.removeById(id);
    }
    
    /**
     * 导出用户 - 记录操作日志
     */
    @OperationLog(module = "用户管理", type = "EXPORT", desc = "导出用户列表")
    @GetMapping("/export")
    public void exportUsers() {
        // 导出逻辑...
    }
}
~~~

## Spring中事务失效的场景

三种情况：

> **异常捕获处理，方法自己捕获异常，没有抛出，解决方法：抛出异常**
>
> **抛出异常检查，解决方法：配置rollbackFor属性为Exception**
>
> **非public方法导致事务失效，解决方法：改为public**

**异常捕获处理**

~~~java
@Transactional
public void 参数名(参数1,2,...){
    try{
        //业务逻辑...
        
        非检查异常		//这里出现异常，由下面catch捕获处理了，事务通知无法知悉
            
        //业务逻辑....
    }catch(Exception e){
        e.printStackTrace();
        //解决方法:抛出异常
        //throw new RuntimeException(e)
    }
}
~~~

**抛出检查异常**

~~~java
//解决方法，配置rollbackFor属性
//@Transactional(rollbackFor=Exception.class)
@Transactional
public void 参数名(参数1,2,...){
    try{
        //业务逻辑...
        
        检查异常		//Spring默认只会回滚非检查异常，这里的不会回滚
            
        //业务逻辑....
    }catch(Exception e){
        e.printStackTrace();
        throw new RuntimeException(e)
    }
}
~~~

**非public方法**

~~~java
@Transactional(rollbackFor=Exception.class)
void 参数名(参数1,2,...){  //解决方法:改为public
    try{
        //业务逻辑...
        
        异常		//Spring为方法创建代理，添加事务通知，前提条件都是该方法是public的
            
        //业务逻辑....
    }catch(Exception e){
        e.printStackTrace();
        throw new RuntimeException(e)
    }
}
~~~

























































