---
标题: 任务规划与多Agent协作
类型: 学习笔记
主题: AI-Agent
学习顺序: 4
状态: 已整理
创建日期: 2026-07-24
更新日期: 2026-07-27
来源: https://github.com/wangyuefan09/agent-zero
来源说明: Agent Zero源码与中文文档（王越凡及项目贡献者）
---

# 任务规划与多Agent协作

## 任务规划是什么

任务规划是把目标转成可执行步骤，并在获得新信息后更新计划。规划可以是：

- 模型在每轮隐式决定下一步；
- 先生成显式 Plan，再逐步执行；
- 用固定 Workflow 控制主路径，局部交给 Agent；
- 把子任务委派给其他 Agent。

规划质量不只看步骤是否合理，还要看是否可执行、可验证、依赖关系是否正确，以及失败后能否恢复。

## 什么时候拆子Agent

适合拆分：

- 子任务目标明确，可以独立验收；
- 子任务会产生大量中间过程；
- 需要不同角色或 Profile；
- 多个子任务相互独立，可以并行；
- 主 Agent 只需要最终结论或 Artifact。

不适合拆分：

- 任务很短，委派成本高于执行成本；
- 子任务强依赖共享 Browser、Shell 或短期状态；
- 结果难以验证；
- 每一步都需要主 Agent 高频协调；
- 子 Agent 权限不清晰。

## 项目串行父子Agent

`tools/call_subordinate.py` 的执行流程：

1. 校验目标 Agent Profile；
2. 创建或复用 subordinate；
3. 建立 superior/subordinate 引用；
4. 把子任务写入子 Agent History；
5. 运行子 Agent `monologue()`；
6. 子任务结束后调用 `new_topic()` 封存当前主题；
7. 将结果作为 Tool Response 返回主 Agent。

子 Agent 拥有独立 History，但串行委派发生在同一个聊天 Context 内。这样可以减少主 History 中的中间细节，又能保留统一日志和用户交互入口。

## 结果回传

自由文本结果简单，但容易缺少状态。更适合复杂项目的结构是：

```json
{
  "status": "completed|partial|failed",
  "summary": "结论",
  "artifacts": ["文件路径或资源ID"],
  "evidence": ["验证结果"],
  "open_issues": ["未解决问题"],
  "next_action": "建议下一步"
}
```

主 Agent 可以据此判断是否验收、重试或继续拆解。

## 并行任务

项目通过 `tools/parallel.py` 和 `helpers/parallel_tools.py` 管理两类并行 Job：

- 子 Agent Worker Context；
- 直接 Tool Job。

支持启动、等待、取消、刷新和清理。子 Agent Worker 会复制父 Context 的 Project 绑定，但使用独立上下文处理任务。

### 并行的约束

- 限制最大并发数；
- 不允许 worker 再递归启动并行，避免指数膨胀；
- 父任务取消时向子任务传播；
- 每个 Job 设置独立超时；
- 明确共享文件的并发写策略；
- 合并结果时保留 Job 与来源映射。

## 上下文隔离的收益

假设三个研究子任务各产生 10,000 Token 过程：

- 单 Agent 全部执行时，主 History 需要承载完整 30,000 Token；
- 子 Agent 分别执行时，主 History 可以只接收三个摘要和 Artifact 引用。

这能降低上下文负担，但总模型 Token 不一定减少，因为每个子 Agent 都有 System Prompt、工具说明和自身循环。多 Agent 的主要收益是专注与隔离，不应默认等同于降本。

## 多Agent常见问题

### 错误拆解

子任务边界不清，多个 Agent 重复工作或遗漏关键依赖。

### 结果不可验证

子 Agent 返回“已经完成”，但没有文件、测试或来源。

### 上下文不一致

父 Agent 更新了目标，已运行的子 Agent 仍按旧目标执行。

### 成本失控

层层委派或同时启动过多 Agent，模型调用快速增长。

### 副作用冲突

多个 Agent 同时修改同一文件、操作同一浏览器或提交同一外部任务。

## 任务预算

生产系统需要统一预算：

- 最大委派深度；
- 最大子任务数；
- 最大并发数；
- 单 Job 和总任务超时；
- 单 Agent 和总 Token；
- 最大工具调用次数；
- 可重试次数；
- 允许的副作用范围。

预算应由运行时强制执行，而不是只写在 Prompt 里。

## 如何评估多Agent

与单 Agent 基线对比：

- 端到端任务成功率；
- 子任务完成率；
- 结果合并正确率；
- 总步骤与重复步骤；
- 总 Token 与总成本；
- P50/P95 耗时；
- 主 Context Token 峰值；
- 并行冲突与取消成功率。

只有质量或耗时收益超过协调成本时，多 Agent 才值得保留。

## 公众号文章

- [[02-公众号/已发布/Agent系列/04-CoT、ReAct和Plan-and-Execute/正文|Agent 是如何思考的？一篇看懂 CoT、ReAct 和 Plan-and-Execute]]

## 关联笔记

- [[01-Agent基础与运行循环|Agent基础与运行循环]]
- [[02-多轮会话与上下文管理|多轮会话与上下文管理]]
- [[03-工具调用与执行系统|工具调用与执行系统]]
- [[10-Agent评估与面试问题清单|Agent评估与面试问题清单]]

## 顺序导航

- 上一节：[[03-工具调用与执行系统|工具调用与执行系统]]
- 下一节：[[05-记忆系统与知识召回|记忆系统与知识召回]]
