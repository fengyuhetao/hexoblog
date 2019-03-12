---
title: java开发小记
abbrlink: 7845
date: 2018-12-27 21:13:11
tags:
---

# 秒杀

链接:https://github.com/fengyuhetao/miaosha.git

## 两次MD5(登录保护)

1. 用户端

   用户输入pass = md5(明文+固定salt)        

2. 服务端

   pass = md5(用户输入pass+随机salt)，随机salt存入数据库

个人感觉: 感觉没啥用，第一次md5用于防止密码直接在公网上传输，只需要将抓取到的md5之后的值作为**伪**密码，不在经过js处理，依然可以正常登录。

## JSR参数验证(自定义注解)

### 定义接口`interface IsMobile`

```
package com.ht.miaosha.validator;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import javax.validation.Constraint;
import javax.validation.Payload;

@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Constraint(
        validatedBy = {IsMobileValidator.class}
)
public @interface IsMobile {
    boolean required() default true;

    String message() default "手机号码格式错误";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

### 定义验证类`IsMobileValidator`

```
package com.ht.miaosha.validator;

import com.ht.miaosha.util.ValidatorUtil;
import org.thymeleaf.util.StringUtils;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

/**
 * Created by hetao on 2018/12/28.
 */
public class IsMobileValidator implements ConstraintValidator<IsMobile, String>{

    private boolean required = false;

    @Override
    public void initialize(IsMobile constraintAnnotation) {
        required = constraintAnnotation.required();
    }

    @Override
    public boolean isValid(String s, ConstraintValidatorContext constraintValidatorContext) {
        if(required) {
            return ValidatorUtil.isMobile(s);
        } else {
            return StringUtils.isEmpty(s) || ValidatorUtil.isMobile(s);
        }
    }
}
```

## 统一异常拦截

```
package com.ht.miaosha.exception;

import com.ht.miaosha.result.CodeMsg;
import com.ht.miaosha.result.Result;
import org.springframework.validation.BindException;
import org.springframework.validation.ObjectError;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

import javax.servlet.http.HttpServletRequest;
import java.util.List;

/**
 * Created by hetao on 2018/12/28.
 */
@ControllerAdvice
@ResponseBody
public class GlobalExceptionHandler {

    @ExceptionHandler(value = Exception.class)
    public Result<String> exceptionHandler(HttpServletRequest request, Exception e) {
        if(e instanceof GlobalException) {
            GlobalException ex = (GlobalException) e;
            return Result.error(ex.getCm());
        } else if(e instanceof BindException) {
            BindException ex = (BindException) e;
            List<ObjectError> errorList = ex.getAllErrors();
            ObjectError error = errorList.get(0);

            String msg = error.getDefaultMessage();
            return Result.error(CodeMsg.BIND_ERROR.fillArgs(msg));
        } else {
            return Result.error(CodeMsg.SERVER_ERROR);
        }
    }
}

```

## 注入参数

### 定义一个resolver

```
package com.ht.miaosha.config;

import com.ht.miaosha.entity.MiaoshaUser;
import com.ht.miaosha.service.impl.MiaoshaUserServiceImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.MethodParameter;
import org.springframework.lang.Nullable;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.thymeleaf.util.StringUtils;

import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * Created by hetao on 2018/12/28.
 */
@Service
public class UserArgumentResolver implements HandlerMethodArgumentResolver {
    @Autowired
    MiaoshaUserServiceImpl userService;

    @Override
    public boolean supportsParameter(MethodParameter methodParameter) {
        Class<?> clazz = methodParameter.getParameterType();
        return clazz == MiaoshaUser.class;
    }

    @Nullable
    @Override
    public Object resolveArgument(MethodParameter methodParameter, @Nullable ModelAndViewContainer modelAndViewContainer, NativeWebRequest nativeWebRequest, @Nullable WebDataBinderFactory webDataBinderFactory) throws Exception {
        HttpServletRequest request = nativeWebRequest.getNativeRequest(HttpServletRequest.class);
        HttpServletResponse response = nativeWebRequest.getNativeResponse(HttpServletResponse.class);
        String paramToken = request.getParameter(MiaoshaUserServiceImpl.COOKIE_NAME_TOKEN);
        String cookieToken = getCookieValue(request, MiaoshaUserServiceImpl.COOKIE_NAME_TOKEN);

        if(StringUtils.isEmpty(cookieToken) && StringUtils.isEmpty(paramToken)) {
            return "login";
        }
        String token = StringUtils.isEmpty(paramToken) ? cookieToken : paramToken;
        return userService.getByToken(response, token);

    }

    private String getCookieValue(HttpServletRequest request, String cookieNameToken) {
        Cookie[] cookies = request.getCookies();
        for (Cookie cookie: cookies) {
            if(cookie.getName().equals(cookieNameToken)) {
                return cookie.getValue();
            }
        }

        return null;
    }
}
```

### 添加到resolves列表

```
package com.ht.miaosha.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;

import java.util.List;

/**
 * 拦截器
 * Created by hetao on 2018/12/28.
 */
@Configuration
public class WebConfig extends WebMvcConfigurationSupport{
    @Autowired
    UserArgumentResolver argumentResolver;

    @Override
    protected void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(argumentResolver);
    }
}

```

## forward和redirect跳转的区别

1、request.getRequestDispatcher("a").forward(rquest,response); request转发，它可以保存request中的数据，页面跳转，但是地址是不调整的 。
2、response.sendRedirect("b"); 方式是重定向，它的数据是不共享的，也就是说 request中保存的数据在b页面中是获取不到的，这种方式是表单是不能重复提交的 ，
3、respons跳转是可以实现跨域的，地址栏也会变化。

## 压测入门

先入门，其他慢慢翻文档。

###  jmeter使用入门

https://www.cnblogs.com/test002/p/8034154.html

### 自定义变量模拟多用户

1. 添加-配置元件-CSV数据文件设置
2. 设置文件名，变量名称比如:`token`
3. 引用变量名称比如 `${token}`

参考链接: https://www.cnblogs.com/jessicaxu/p/7512680.html

### 命令行使用

1. 录制好jmx
2. sh jmeter.sh -n -t xxx.jmx -l result.jtl
3. 将result.jtl 导入到jmeter

### Redis压测工具Redis-benchmark

1. redis-benchmark -h 127.0.0.1 -p 6379 -c 100 -n 100000               100个并发连接，100000个请求
2. redis-benchmark -h 127.0.0.1 -p 6379 -q -d 100                             存取大小为100字节的数据包

资料比较多，直接找就行

## springboot打包war包

1. 添加spring-boot-starter-tomcat的provided依赖
2. 添加maven-war-plugin插件

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-tomcat</artifactId>
	<scope>provided</scope>
</dependency>

<build>
	<finalName>${project.artifactId}</finalName>
	<plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-war-plugin</artifactId>
            <configuration>
            	<failOnMissingWebXml>false</failOnMissingWebXml>
            </configuration>
        </plugin>
    </plugins>
</build>
```

处理一下入口类:

```

@SpringBootApplication
public class MiaoshaApplication extends SpringBootServletInitializer {

	public static void main(String[] args) {
		SpringApplication.run(MiaoshaApplication.class, args);
	}

	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
		return builder.sources(MiaoshaApplication.class);
	}
}

```

最后:

`mvn clean package`

## 页面优化技术

1. 页面缓存+URL缓存+对象缓存

   * Cache Aside Pattern

   * https://blog.csdn.net/goldenfish1919/article/details/79739186
   * spring.resources 系列配置         具体慢慢了解

2. 页面静态化，前后端分离

   * 尽量使得前端页面没有变化          利用浏览器缓存
   * spring.resources 系列配置         具体慢慢了解

3. 静态资源优化

   * js/css 压缩                     删掉空白字符和注释等等，从而减少流量               Tengine 支持该功能             webpack用于打包
   * 多个js/css组合              减少连接数            Tengine 支持该功能

4. CDN优化

   * 就近访问需要在了解

##  超卖问题

数据库添加唯一索引：防止重复购买

sql加库存量判断：防止库存变成负数

 

秒杀项目流程:

1. 秒杀控制器初始化阶段，将所有秒杀商品的库存量保存到redis里边，并设置秒杀商品未被秒杀完成，使用hashmap保存。 

2. 首先从hashmap读取该商品是否秒杀完成，（可以减少redis访问次数）如果完成，结束，否则转下一步

3. 如果没有完成，验证path。失败，结束，否则转下一步

4. 从redis对应商品id的set（如果可以购买多个，则加上数量，或者使用list）中查询用户是否存在，如果存在，说明重复点击，结束，如果不存在

5. redis减库存，并将用户添加到set中，如果库存变为负，判定该商品秒杀结束。（如果，不添加到set中的话，如果同一个用户连续点13次，则会出现redis值为负的问题，实际上，只有一个用户秒杀成功的情况）

6. 将该消息进入消息队列

7. 读取库存，判断库存是否小于0，小于0，结束，否则转下一步

8. 判断用户是否已经秒杀到商品，如果是，结束，否则转下一步

9. 创建一个秒杀事务

   1. 数据库中减少库存，库存减1，如果失败，在内存中设置秒杀结束，否则下一步

      (update table_name set stock=stock-1 where stock >= 1 and good_id = xxx);

   2. 创建一条订单信息，也是事务

      1. 创建一条新的订单
      2. 在秒杀订单表中，添加一条记录
      3. 在redis中添加用户秒杀的记录


## 接口优化

1. Redis预减库存减少数据库访问
   1. 系统初始化，将商品库存数量加载到redis
   2. 收到请求，redis预减库存，库存不足，直接返回，否则转3
   3. 请求入队，立即返回排队中 
   4. 请求出队，生成订单，减少库存
   5. 客户端轮询，是否秒杀成功
2. 内存标记减少redis访问
3. 请求先入队缓冲，异步下单，增强用户体验
4. RabbitMQ + Spring Boot
5. nginx水平扩展
6. 分库分表           工具：mycat

## 安全优化

1. 秒杀地址隐藏
2. 数学公式验证码
3. 接口限流防刷 



