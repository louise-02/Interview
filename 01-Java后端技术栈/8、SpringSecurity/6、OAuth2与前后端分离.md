# 6、OAuth2 与前后端分离

> 本章是 **上 SSO / 统一认证中心** 时再读；单体 JWT 集成见 **[0、生产集成完整方案](./0、生产集成完整方案.md)**。

本章理清 **OAuth2/OIDC 几个角色** 与 Spring Security 模块对应关系；完整 Authorization Server 搭建篇幅大，生产常对接 **Keycloak / 阿里云 IDaaS / 自建 AS**。

---

## 1、OAuth2 角色

| 角色 | 说明 |
| ---- | ---- |
| **Resource Owner** | 用户本人 |
| **Client** | 前端或后端应用（申请 Token） |
| **Authorization Server** | 发 Token（登录页、授权码） |
| **Resource Server** | 受保护 API（校验 Access Token） |

Spring 模块：

| 依赖 | 角色 |
| ---- | ---- |
| `spring-boot-starter-oauth2-client` | 作为 Client 对接第三方登录 |
| `spring-boot-starter-oauth2-resource-server` | 作为 Resource Server 校验 JWT |
| `spring-authorization-server` | 自建 Authorization Server（Spring 官方，替代旧 Spring Security OAuth） |

---

## 2、常见场景选型

| 场景 | 方案 |
| ---- | ---- |
| 自有账号 + JWT | 登录接口自签 JWT（[4、JWT](./4、JWT与ResourceServer.md)） |
| API 只校验 Token | Resource Server |
| 微信/GitHub/企业 SSO 登录 | OAuth2 Client + 回调 |
| 统一认证中心 | 独立 Authorization Server 或 Keycloak |

---

## 3、OAuth2 Client（第三方登录）概要

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: xxx
            client-secret: xxx
            scope: read:user
        provider:
          github:
            authorization-uri: https://github.com/login/oauth/authorize
            token-uri: https://github.com/login/oauth/access_token
            user-info-uri: https://api.github.com/user
```

浏览器走授权码流程，回调后建立 Session 或换本地 JWT（视架构而定）。

---

## 4、与单体 JWT 方案对比

| | 自签 JWT | OAuth2 + 统一 AS |
| - | -------- | ---------------- |
| 复杂度 | 低 | 高 |
| 多系统 SSO | 需自己打通 | AS 天然支持 |
| Token 规范 | 自定义 | 标准 OIDC/JWT |
| 适用 | 单体、内部系统 | 多应用、企业 SSO |

---

## 5、Gateway + OAuth2

- Gateway 作为 **Resource Server** 统一验 Token，下游内网可简化（需防绕过 Gateway）。  
- 或 Gateway 只做路由，**每个服务各自 Resource Server**（零信任，重复校验）。  

与 [SpringCloud/实践/3、Gateway鉴权](../../SpringCloud/实践/3、Gateway鉴权与Feign透传Token.md) 结合阅读。

---

## 6、Spring Authorization Server（了解）

Spring 官方新授权服务器项目，替代已停更的 `spring-security-oauth2`。  
若需自建登录中心，按官方 Sample + `RegisteredClient`、`AuthorizationServerSettings` 配置；细节随版本迭代快，以官方文档为准。

本笔记不展开完整 AS 部署，需要时再单独开文档。

---

## 7、安全建议

- 授权码 + PKCE（公网前端）  
- Client Secret 不进前端  
- Refresh Token 放 HttpOnly Cookie 或安全存储  
- Scope 最小权限  
- 生产 HTTPS
