# 1、与 Activiti 工作流鉴权

工作流完整集成见 **[Activiti/0、生产集成完整方案](../../9、Activiti/0、生产集成完整方案.md)**，身份衔接见 **[Activiti/5、与Spring Security集成](../../9、Activiti/5、与SpringSecurity集成.md)**。本章列 Security 侧必记点。

---

## 1、唯一身份来源

assignee / 发起人 = **`SecurityUtils.getUsername()`**（或项目统一的 userId 字符串）。  
HTTP 线程内 Controller 与 Flowable/Activiti API 读到同一 `SecurityContext`。

---

## 2、三件事必做

| 项 | 说明 |
| -- | ---- |
| 启动流程 | 变量 `applyUserId` = 当前用户 |
| 查待办 | assignee + candidateUser + candidateGroup（组名与 `ROLE_*` 对齐） |
| 办理任务 | **`TaskPermissionService.canHandle`**，禁止裸 `complete` |

---

## 3、异步节点

Job / 定时器线程无 JWT → 用系统身份 `SecurityContextHolder.setAuthentication` 或 `DelegatingSecurityContext*`，见 [Activiti/5、与SpringSecurity集成 §6](../../9、Activiti/5、与SpringSecurity集成.md)。

---

## 4、Gateway 场景

Gateway 验 JWT → 业务服务 Filter 恢复 Context → 再调 `/api/workflow/**`。Feign 透传 Token 见 [Gateway 实践](../../7、SpringCloud/实践/3、Gateway鉴权与Feign透传Token.md)。
