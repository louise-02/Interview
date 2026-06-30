# Activiti / 工作流

基于 **Spring Boot 3 + Flowable 7**（与 Activiti **同源 API**；老项目换包名即可）。

> **从这里开始** → **[0、生产集成完整方案](./0、生产集成完整方案.md)**  
> 先看**生产可用的一整套**：依赖、BPMN、类清单、Controller、与 Security 衔接；跑通后再读 1～5 章原理。

**前置**：[Spring Security 生产集成](../8、SpringSecurity/0、生产集成完整方案.md)（JWT 登录已通）。

## 本模块内容

| 类型 | 入口 |
| ---- | ---- |
| **生产集成（首选）** | **[0、生产集成完整方案](./0、生产集成完整方案.md)** |
| **原理详解** | [1、引擎概述与架构](./1、引擎概述与架构.md) · [2、BPMN与流程定义](./2、BPMN与流程定义.md) · [3、API与运行时原理](./3、API与运行时原理.md) · [4、节点扩展Delegate与Listener](./4、节点扩展Delegate与Listener.md) · [5、与SpringSecurity集成](./5、与SpringSecurity集成.md) |
| **速查** | [7、知识体系总览](./7、知识体系总览.md) |
| **实践** | [0、常用片段与踩坑](./实践/0、常用片段与踩坑.md) |

## 推荐阅读顺序

```
0、生产集成完整方案（请假流程 BPMN + TaskAppService + 鉴权）
    ↓ 能启动、待办、审批 complete
1～5 章（按需深入）
7、知识体系总览（复习）
```

## 相关

| 模块 | 链接 |
| ---- | ---- |
| Spring Security | [生产集成](../8、SpringSecurity/0、生产集成完整方案.md) · [实践/1、工作流鉴权要点](../8、SpringSecurity/实践/1、与Activiti工作流鉴权.md) |
| 策略模式（节点内扩展） | [14、策略模式](../../04-设计模式/14、策略模式.md) |
| Gateway | [Gateway 鉴权](../7、SpringCloud/实践/3、Gateway鉴权与Feign透传Token.md) |

## 怎么写新笔记

- 生产模板变更 → **0、生产集成完整方案**
- BPMN / API 原理 → **1～3**
- Delegate / 策略 → **4**
- 身份与 Security 衔接 → **5**
- 踩坑 → **实践/0**
