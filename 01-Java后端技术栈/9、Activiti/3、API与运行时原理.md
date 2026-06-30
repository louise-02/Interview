# 3、API 与运行时原理

> HTTP 封装模板见 **[0、生产集成完整方案](./0、生产集成完整方案.md)** §7。

## 1、启动流程

```java
ProcessInstance pi = runtimeService.startProcessInstanceByKey(
    "leave_process",           // 流程定义 key
    "LEAVE-20250612-001",      // businessKey
    Map.of("applyUserId", "zhangsan", "days", 3));
```

启动后：

1. `ACT_RU_EXECUTION` 插入执行流  
2. 第一个 User Task 若配置了 assignee，**ACT_RU_TASK** 出现待办  
3. 变量写入 `ACT_RU_VARIABLE`  

---

## 2、查询待办

```java
List<Task> list = taskService.createTaskQuery()
    .taskAssignee(userId)
    .orderByTaskCreateTime().desc()
    .list();
```

候选人任务：

```java
.taskCandidateUser(userId)
// 或
.taskCandidateGroupIn(List.of("dept_manager"))
```

**or 组合**见生产方案 `TaskAppService.todoList`。

---

## 3、签收与完成

| 操作 | API | 场景 |
| ---- | --- | ---- |
| 签收 | `taskService.claim(taskId, userId)` | candidate 任务变为 assignee |
| 完成 | `taskService.complete(taskId, variables)` | 流转下一节点 |
| 委派 | `taskService.delegateTask` | 临时转交 |
| 转办 | `taskService.setAssignee` | 直接改办理人 |

`complete` 传入的 variables 会合并进流程实例，供后续网关、节点表达式使用。

---

## 4、流程变量作用域

| 方法 | 作用域 |
| ---- | ------ |
| `runtimeService.setVariable(processInstanceId, name, value)` | 整个实例 |
| `taskService.setVariableLocal(taskId, name, value)` | 仅当前任务 |

网关条件、Service Task 一般用**实例级**变量。

---

## 5、历史与轨迹

```java
historyService.createHistoricProcessInstanceQuery()
    .processInstanceBusinessKey(businessKey)
    .singleResult();

historyService.createHistoricTaskInstanceQuery()
    .processInstanceId(processInstanceId)
    .orderByHistoricTaskInstanceEndTime().asc()
    .list();
```

用于「审批记录」时间线 UI。

---

## 6、删除与挂起

```java
runtimeService.deleteProcessInstance(processInstanceId, "用户撤销");
runtimeService.suspendProcessInstanceById(processInstanceId);
```

删除会清运行表，历史仍可在 `ACT_HI_*` 查到（视配置）。

---

## 7、常见坑

| 现象 | 原因 |
| ---- | ---- |
| 启动后无待办 | assignee 表达式变量未设；或节点是 Service Task |
| complete 后流程不动 | 网关条件不满足；下一节点异步 Job 未执行 |
| 查不到 candidate 任务 | 组名与 RBAC 不一致；未 claim 且只查 assignee |
| 变量 ClassCastException | 表达式期望 Boolean，传了字符串 `"true"` |
