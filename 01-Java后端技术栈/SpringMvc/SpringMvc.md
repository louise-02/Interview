# Spring MVC

Spring MVC 是 Spring Framework 的 **Web 模块**，实现 MVC 模式，处理 HTTP 请求、参数绑定、视图/JSON 响应。

## 目录

| 章节 | 内容 |
| ---- | ---- |
| [1、架构与请求流程](./1、架构与请求流程.md) | DispatcherServlet、九大组件、一次请求全流程 |
| [2、Controller 与参数绑定](./2、Controller与参数绑定.md) | 注解、校验、JSON、文件上传 |
| [3、拦截器过滤器与跨域](./3、拦截器过滤器与跨域.md) | Filter vs Interceptor、CORS |

## 与 Spring Boot 关系

Boot 自动配置 `DispatcherServlet`、默认 JSON（Jackson）、内嵌 Tomcat。工程实践见 [SpringBoot.md](../SpringBoot/SpringBoot.md)（拦截器、参数解析器等代码）。
