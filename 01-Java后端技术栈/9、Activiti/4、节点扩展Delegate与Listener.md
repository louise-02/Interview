# 4、节点扩展 Delegate 与 Listener

> Delegate Bean 示例见 **[0、生产集成完整方案](./0、生产集成完整方案.md)** §7.6。

## 1、JavaDelegate（Service Task）

BPMN：

```xml
<serviceTask id="notify" name="发通知"
             flowable:delegateExpression="${notifyManagerDelegate}"/>
```

Java：

```java
@Component("notifyManagerDelegate")
public class NotifyManagerDelegate implements JavaDelegate {
    @Override
    public void execute(DelegateExecution execution) {
        String bk = execution.getProcessInstanceBusinessKey();
        Object approved = execution.getVariable("approved");
        // 调消息、写库
    }
}
```

也可用 `flowable:class="com.example.MyDelegate"`（全类名，不利于 Spring 注入，**推荐 delegateExpression**）。

---

## 2、TaskListener

挂在 User Task 生命周期：

```xml
<userTask id="managerTask" name="主管审批">
  <extensionElements>
    <flowable:taskListener event="create" delegateExpression="${managerTaskCreateListener}"/>
  </extensionElements>
</userTask>
```

| event | 时机 |
| ----- | ---- |
| create | 任务创建（待办生成） |
| assignment | assignee 变化 |
| complete | 完成前 |
| delete | 删除 |

用途：发待办通知、写审批日志、从业务表填变量。

---

## 3、ExecutionListener

流程/节点进入离开：

```xml
<endEvent id="endApproved">
  <extensionElements>
    <flowable:executionListener event="end" delegateExpression="${processEndListener}"/>
  </extensionElements>
</endEvent>
```

适合：流程结束时统一改业务单状态。

---

## 4、策略模式扩展节点逻辑

同一 Service Task 多种业务算法时，不要堆 if-else：

```java
public interface LeaveRuleStrategy {
    boolean support(String leaveType);
    int calcDays(Map<String, Object> vars);
}

@Service
@RequiredArgsConstructor
public class LeaveRuleDelegate implements JavaDelegate {
    private final List<LeaveRuleStrategy> strategies;

    @Override
    public void execute(DelegateExecution execution) {
        String type = (String) execution.getVariable("leaveType");
        LeaveRuleStrategy strategy = strategies.stream()
            .filter(s -> s.support(type))
            .findFirst()
            .orElseThrow();
        execution.setVariable("calcDays", strategy.calcDays(execution.getVariables()));
    }
}
```

BPMN 只负责**编排**；节点内用 **策略** 选实现，见 [14、策略模式](../../04-设计模式/14、策略模式.md)。

---

## 5、异步与定时

```xml
<serviceTask id="asyncNotify" flowable:async="true"
             flowable:delegateExpression="${notifyManagerDelegate}"/>

<intermediateCatchEvent id="timer">
  <timerEventDefinition>
    <timeDuration>PT24H</timeDuration>
  </timerEventDefinition>
</intermediateCatchEvent>
```

需 `flowable.async-executor-activate: true`。异步线程**没有** HTTP SecurityContext，见 [5、与SpringSecurity集成](./5、与SpringSecurity集成.md) §6。

---

## 6、Spring 事务

Delegate / Listener 内调业务库，默认与 Flowable 同一事务边界（Spring 集成下）。远程调用或 MQ 建议：

- 短事务内只写必要状态  
- 或 `TransactionSynchronization.afterCommit` 再发消息  

避免引擎回滚了但消息已发出。
