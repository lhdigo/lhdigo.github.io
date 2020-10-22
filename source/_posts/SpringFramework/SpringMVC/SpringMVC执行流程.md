---
title: SpringMVC执行流程
date: 2020-10-10 23:19:09
tags: SpringMVC
categories: [SpringFramework,SpringMVC]
---

# SpringMVC执行流程图

![SpringMVC执行流程图](https://i.loli.net/2020/10/19/SytjFUTe3nJiWEm.jpg)

<!--more-->

# SpringMVC执行流程

1. 用户发送请求至前端控制器 **DispatcherServlet。**
2. **DispatcherServlet** 收到请求调用 **HandlerMapping** 处理器映射器。
3. 处理器映射器 **HandlerMapping** 找到具体的处理器(可以根据xml配置、注解进行查找)，生成处理器对象及处理器拦截器(如果有则生成)一并返回给 **DispatcherServlet**。
4. 前端控制器 **DispatcherServlet。**请求**HandlerAdapter**执行**Handler**
5.  **HandlerAdapter** 经过适配调用具体的处理器(Controller，也叫后端控制器)。
6.  Controller执行完成返回 **ModelAndView**。
7. **HandlerAdapter** 将controller执行结果 **ModelAndView** 返回给 **DispatcherServlet**。
8. **DispatcherServlet** 将 **ModelAndView** 传给 **ViewReslover** 视图解析器。
9. **ViewReslover** 解析后返回具体View。
10. **DispatcherServlet** 根据View进行渲染视图（即将模型数据填充至视图中）。
11. **DispatcherServlet** 响应用户。