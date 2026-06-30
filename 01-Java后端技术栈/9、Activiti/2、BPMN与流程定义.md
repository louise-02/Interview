# 2、BPMN 与流程定义

> 完整请假 BPMN 见 **[0、生产集成完整方案](./0、生产集成完整方案.md)** §6。

## 1、常用元素

| 元素 | 用途 |
| ---- | ---- |
| Start Event | 开始 |
| User Task | 人工办理（待办） |
| Service Task | 自动执行（JavaDelegate、HTTP 等） |
| Exclusive Gateway | 排他分支（if-else） |
| Parallel Gateway | 并行 |
| End Event | 结束 |

---

## 2、User Task 三种办理人

| 配置 | 含义 | 查询 |
| ---- | ---- | ---- |
| `assignee="zhangsan"` | 固定办理人 | `taskAssignee("zhangsan")` |
| `assignee="${applyUserId}"` | 表达式，启动时设变量 | 变量值即用户 ID |
| `candidateUsers` / `candidateGroups` | 候选人/组，需 claim | `taskCandidateUser` / `taskCandidateGroupIn` |

**生产建议**：

- 申请人节点：`assignee="${applyUserId}"`，启动流程时 `vars.put("applyUserId", currentUser)`  
- 审批节点：`candidateGroups="dept_manager"`，与系统角色组名一致  

---

## 3、表达式

Flowable 使用 **UEL**，写法 `${variableName}`、`${days > 3}`。

网关条件示例：

```xml
<conditionExpression xsi:type="tFormalExpression"><![CDATA[${approved == true}]]></conditionExpression>
```

User Task 动态办理人：

```xml
<userTask id="managerTask" flowable:assignee="${deptManagerId}"/>
```

`deptManagerId` 由上一节点 Service Task 或 Listener 写入变量。

---

## 4、流程定义部署

| 方式 | 说明 |
| ---- | ---- |
| classpath 自动部署 | `processes/*.bpmn20.xml` + `check-process-definitions: true` |
| API 部署 | `repositoryService.createDeployment().addClasspathResource(...).deploy()` |
| 设计器导出 | Camunda Modeler / Flowable Modeler 导出 XML |

**注意**：已运行实例绑定**部署版本**；改 BPMN 会生成新 ProcessDefinition，老实例仍跑旧版。

---

## 5、process id 与 key

- XML 里 `<process id="leave_process">` → **processDefinitionKey**  
- 代码：`startProcessInstanceByKey("leave_process", businessKey, variables)`  

name 仅展示；**id/key 才是代码用的标识**。

---

## 6、BPMN 与业务表

推荐每个流程实例带 **businessKey**（业务单号）：

```java
runtimeService.startProcessInstanceByKey("leave_process", "LEAVE-20250612-001", vars);
```

查询：`createProcessInstanceQuery().processInstanceBusinessKey(businessKey)`。

业务表存 `process_instance_id` + `business_key` 双向关联。
