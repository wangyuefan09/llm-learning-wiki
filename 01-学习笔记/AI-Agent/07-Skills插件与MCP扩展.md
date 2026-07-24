---
标题: Skills、Plugin与MCP扩展
类型: 学习笔记
主题: AI-Agent
学习顺序: 7
状态: 已整理
创建日期: 2026-07-24
更新日期: 2026-07-24
来源: https://github.com/wangyuefan09/agent-zero
来源说明: Agent Zero源码与中文文档（王越凡及项目贡献者）
---

# Skills、Plugin与MCP扩展

## 四种扩展方式

| 机制 | 主要内容 | 适合目标 |
| --- | --- | --- |
| Tool | 可执行函数与参数Schema | 增加一个动作 |
| Skill | Markdown流程、说明和资源 | 教Agent如何完成一类任务 |
| Plugin | Hook、Tool、Prompt、API、配置和UI能力包 | 交付完整模块 |
| MCP | 标准化外部工具Server | 跨进程或远程接入业务工具 |

Agent Profile 还可以覆盖角色、Prompt、Tools 和 Extensions，改变某类 Agent 的整体工作方式。

## Skills

Skill 通常包含 `SKILL.md` 和可选脚本、模板或参考资料。它不是一个直接执行的函数，而是一套可按需加载的操作方法。

项目支持：

- 从全局、Project、Agent Profile 和 Plugin Scope 扫描 Skill；
- 解析 Frontmatter 中的名称与描述；
- 搜索相关 Skill；
- 通过 `skills_tool` 加载正文或读取资源文件；
- 记录已加载 Skill；
- History 压缩后重新附着 Skill Body；
- 控制最大活动 Skill 数量。

### 渐进式披露

如果把所有 Skill 正文都放入 System Prompt，上下文会迅速增长。更合理的流程是：

```text
先提供名称和短描述
→ 根据当前任务搜索候选Skill
→ Agent选择并加载完整SKILL.md
→ 任务结束后按作用域卸载
```

## Plugin

Plugin 可以包含：

```text
plugin.yaml
default_config.yaml
hooks.py
tools/
prompts/
extensions/python/
extensions/webui/
api/
webui/
skills/
```

它适合 Browser、Memory、Desktop 这类同时影响后端、Prompt、工具和 UI 的能力。

## Extension机制

项目提供两类扩展点：

### 显式生命周期扩展

例如：

- `agent_init`；
- `monologue_start/end`；
- `message_loop_start/end`；
- `message_loop_prompts_before/after`；
- `chat_model_call_before/after`；
- `tool_execute_before/after`；
- `hist_add_before`；
- `webui_ws_event`。

### 函数级扩展

`@extensible` 装饰器会为函数自动暴露 start/end 扩展点，扩展可以修改参数、短路返回或处理异常。这提高灵活性，但也使调用链更隐式，需要日志、命名约定和测试保证可维护性。

## MCP

MCP 把外部 Server 的工具 Schema 和调用结果接入 Agent。项目支持：

- command/stdio 本地 Server；
- URL 远程 Server；
- headers、env、disabled tools；
- 全局与 Project 配置合并；
- 工具发现与 Prompt 生成；
- 文本、图片、音频、Resource 的结果适配；
- 初始化、状态、日志、超时与刷新。

MCP Tool 最终适配成 `helpers.tool.Tool`，因此执行前后钩子、日志和 History 回填与本地工具一致。

## 如何选择扩展方式

- 只需执行一个明确动作：Tool；
- 已有工具但需要固定方法：Skill；
- 需要完整后端与UI模块：Plugin；
- 业务能力独立部署或跨语言：MCP；
- 需要整体改变Agent工作方式：Agent Profile。

这些机制可以组合。例如 Browser 是 Plugin，内部提供 Tool 和 Skills；一个企业 CRM 可以作为 MCP Server，再由销售 Skill 描述调用顺序。

## 版本与兼容性

扩展系统需要关注：

- Tool Schema 变化；
- Prompt 文件名与加载顺序；
- Extension Hook 参数；
- Project/Global Scope 合并；
- Plugin 配置迁移；
- MCP 协议和 SDK 版本；
- Skill Frontmatter 校验；
- 前后端事件兼容。

运行时 Prompt 文件不是普通文档。新增或翻译 Profile Prompt 可能直接改变工具调用行为，必须进行回归测试。

## 扩展测试

最小验收包括：

1. 能发现并加载；
2. 配置作用域正确；
3. Tool Schema 合法；
4. Prompt 不重复或冲突；
5. Hook 顺序稳定；
6. 禁用/卸载后没有残留；
7. Project 切换不串配置；
8. MCP 断连、超时和返回多媒体时可处理；
9. 日志和错误不泄露 Secrets；
10. History 压缩后 Skill 仍保持需要的指令。

## 关联笔记

- [[03-工具调用与执行系统|工具调用与执行系统]]
- [[06-项目隔离与配置管理|项目隔离与配置管理]]
- [[08-浏览器桌面与文档协作|浏览器、桌面与文档协作]]
- [[09-可观测性恢复与安全边界|可观测性、恢复与安全边界]]

## 顺序导航

- 上一节：[[06-项目隔离与配置管理|项目隔离与配置管理]]
- 下一节：[[08-浏览器桌面与文档协作|浏览器、桌面与文档协作]]
