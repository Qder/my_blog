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
# 四、编写一个`GET /api/auth/me`接口

**作用：客户端带 token 请求这个接口，服务端从 token 里解析出"你是谁"，返回你的信息**

1、创建 `CurrentUserVO`，用于返回当前登录用户的信息。不直接返回 `Student` 实体，因为实体里有 `password`、`deleted` 等不该暴露的字段，是一种安全保护的措施。

2、在 `StudentService` 声明 `getCurrentUser(Long studentId)`，在 `StudentServiceImpl` 实现：按 id 查学生，查不到抛 `STUDENT_NOT_FOUND`，查到则把实体字段拷贝到 `CurrentUserVO` 返回。

3、在 `AuthController` 里新增 `/me` 接口，从 `UserContext` 取当前用户 id，交给 Service 查询并返回。

```
① 请求：GET /api/auth/me，Header 带 Authorization: Bearer xxx
        ↓
② JwtInterceptor.preHandle()：校验 token，UserContext.setCurrentUserId(studentId)
        ↓
③ AuthController.me()：UserContext.getCurrentUserId() 拿到 userId
        ↓
④ StudentServiceImpl.getCurrentUser()：查库、转 VO
        ↓
⑤ 返回 ApiResponse<CurrentUserVO>
        ↓
⑥ JwtInterceptor.afterCompletion()：UserContext.clear()
```

# 五、让业务接口绑定当前登录用户

1、修改 `RegistrationRequest`，删掉 `studentId` 字段。因为 `studentId` 从 token 里取，前端不需要传。

2、修改 `RegistrationController`，`register` 和 `cancel` 都从 `UserContext` 取当前用户 id，不再依赖前端传参。

3、修改 `RegistrationService` 接口，`cancel` 方法加一个 `currentUserId` 参数。

4、修改 `RegistrationServiceImpl` 的 `cancel` 方法，在原子取消之前，先校验报名记录是否属于当前用户。

5、修改测试。所有 `cancel(regId)` 的调用改成 `cancel(regId, studentId)`，参数配对。


