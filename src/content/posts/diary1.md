---
title: diary1
published: 2026-10-11
description: ''
image: ''
tags: []
category: ''
draft: false 
lang: ''
---

2026.10.11

计划为event_hub项目增加密码加密功能

# 一、先引入` BCrypt` 密码加密，学生创建时存储哈希密码

1、为student实体增加密码字段

2、增加`PasswordConfig`文件，使用`spring-security-crypto`依赖下的`passwordEncoder`

# 二、引入`JWT`工具类，以及登录接口的实现

1、在`util`包下面创建`JwtUtil`类，这里会实现生成签名密钥，生成token，解析token，从token中获取`userid`，校验token是否有效。

2、在`application.yml`里面配置`jwt`签名密钥。

3、编写登录接口，登录接口必不可少的就是`DTO`，`VO`两个东西了，前者是`Controller `接收前端请求参数，封装成 `DTO` 传给 Service；Service 查询多个表后，组装成` DTO` 返回给` Controller`，后者是通常由 `Controller `层把 `DTO`或 `Entity` 转换成 `VO` 后返回给前端

4、在service层声明登录接口，先查学号，学号不存在就报错，如果该学号存在就校验密码，再检查学号是否是被停用，避免和密码校验搞混错误码，最后生成token。

5、在controller层创建一个`http`接口，让前端能调用。

# 三、编写token拦截器

作用：它保证只有带合法 token 的请求才能访问受保护的接口，防止未登录用户乱调接口。

1、创建`Usercontext`	,是一个工具类，基于`ThreadLocal`来获取以及存储当前请求的用户的信息，大致流程如下：

```
① 请求进来（带 token）
        ↓
② 拦截器：解析 token → 拿到 userId
        ↓
③ 拦截器：UserContext.setCurrentUserId(userId)
        ↓
④ Controller → Service → Mapper 里任意位置调用 UserContext.getCurrentUserId()
        ↓
⑤ 请求处理完毕，拦截器的 afterCompletion() 里 UserContext.clear()
```

2、创建`JwtInterceptor`，首先放行 OPTIONS 预检请求（`CORS `用），然后从Header中获取token，在校验token（里面有用户id），在将`id`写入`THreadLocal`，最后就是写`json`响应了。

3、创建**`WebMvcConfig`** ，将刚才写的拦截器注册到`springMVC`的拦截链中。
