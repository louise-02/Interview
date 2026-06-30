# 5、与 Spring Security 集成

> 工作流侧代码见 **[0、生产集成完整方案](./0、生产集成完整方案.md)** §7.2～7.5。  
> 登录/JWT/Filter 见 [Spring Security 生产集成](../8、SpringSecurity/0、生产集成完整方案.md)；Security 侧要点见 [实践/1、与Activiti工作流鉴权](../8、SpringSecurity/实践/1、与Activiti工作流鉴权.md)。

## 1、原则

**唯一身份来源**：Spring Security `SecurityContext`。  
工作流 assignee / 发起人 = `SecurityUtils.getUsername()`（或 userId，全项目统一字符串规则）。

不要在 Activiti IdentityService 里单独做一套登录。

---

## 2、当前用户用法

```java
String userId = SecurityUtils.getLoginUser().getUsername();

runtimeService.startProcessInstanceByKey("leave_process", businessKey,
    Map.of("applyUserId", userId));

taskService.createTaskQuery().taskAssignee(userId).list();
```

HTTP 请求线程内，Controller 与引擎 API 读到**同一用户**（JWT Filter 写入 `SecurityContext` 之后）。

---

## 3、candidateGroups 与 RBAC

| BPMN | Spring Security |
| ---- | --------------- |
| `candidateGroups="dept_manager"` | `ROLE_DEPT_MANAGER` → 解析为 `dept_manager` |
| `candidateGroups="finance"` | `ROLE_FINANCE` 或 authority `finance` |

`TaskPermissionService.resolveGroups` 去掉 `ROLE_` 前缀并与 BPMN 组名对齐（见生产方案 §7.2）。

任务查询必须同时查 **assignee + candidateUser + candidateGroup**。

---

## 4、办理越权防护

**禁止**只凭 taskId 直接 `complete`，必须：

```java
if (!taskPermissionService.canHandle(taskId, userId, groups)) {
    throw new AccessDeniedException("无权办理");
}
```

URL 级 `anyRequest().authenticated()` 只保证已登录；**任务级**必须 `canHandle`。  
`@PreAuthorize("hasAuthority('xxx')")` 可做业务细权限；仅写 `isAuthenticated()` 与 URL 规则重复，可省略。

---

## 5、IdentityService（可选）

| 方案 | 说明 |
| ---- | ---- |
| **轻量（推荐）** | 不同步用户；组名与 Spring Security `ROLE_*` 一致；BPMN 只用 candidateGroup |
| **重量** | 登录/定时 sync 用户到 `ACT_ID_*`，设计器里可点选 |

---

## 6、异步 / 定时 / Job 线程

定时器、async Service Task 在**引擎线程池**执行，无 JWT、无 SecurityContext。

```java
Authentication systemAuth = new UsernamePasswordAuthenticationToken(
    "system", null, List.of(new SimpleGrantedAuthority("ROLE_SYSTEM")));
SecurityContextHolder.getContext().setAuthentication(systemAuth);
try {
    // 调 Activiti / 业务库
} finally {
    SecurityContextHolder.clearContext();
}
```

或使用 `DelegatingSecurityContextRunnable` 包装。

---

## 7、Gateway + 微服务

```
客户端 → Gateway 验 JWT → 业务服务 JwtAuthenticationFilter 恢复 SecurityContext → 调 /api/workflow/**
```

Feign 调其它服务需透传 `Authorization` 头，见 [Gateway 实践](../7、SpringCloud/实践/3、Gateway鉴权与Feign透传Token.md)。网关与业务服务 **JWT 密钥须一致**。

---

## 8、流程实例查看权限

除任务办理外，还需控制「谁能看流程详情/审批记录」：

- 发起人、已参与审批人、管理员  
- Service 层按 `businessKey` 查业务表 owner，或查 `HistoricTaskInstance` 是否包含当前 userId  

不要只藏前端按钮，**接口必须校验**。
