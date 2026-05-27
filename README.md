# Civil-X

> **从物理感知到决策建议** —— AI 驱动的土木工程智能分析平台

Civil-X 让土木工程师通过自然语言与 AI 协作：上传图纸、描述需求，AI 自动完成解析、知识检索、有限元仿真，生成分析报告。

![Civil-X 系统架构](assets/civil-x.png)

## 路线图

### P0 · Runtime Gateway 基础设施

**Runtime Gateway 中间层设计**
统一抽象层，适配不同 runtime。opencode 不同版本 = 不同 runtime。
- 协议设计：submit / status / cancel / result
- 接口定义：REST + SSE 双通道
- opencode 适配器：1.3.13 / 1.15.10 两套
- DB schema：runtime 注册、版本管理、会话路由

**统一 SSE 规范**
一套 SSE 协议，Gateway → BFF → 前端，承载 token 流 + MCP 任务事件 + 停止信令。
- 事件类型枚举：token / tool_call / task_event / error / stop / cancel_ack
- BFF SSE 层改造（不再直连 sidecar）
- 前端 SSE 客户端适配

### P1 · 当前迭代

**UAT 环境迁移启用**
锁定周五版本 → 打 tag → CNB 构建 → 部署到 TKE uat namespace。

**用户 token 计量计费/限额**
用量统计 + 配额控制。计量模型（input/output token × 模型单价）→ DB → BFF middleware → 限额拦截 → 前端展示。

**前端停止对话事件透传**
点停止 → SSE 发 stop 信令 → 真正 cancel 底层调用。覆盖 opencode LLM abort + MCP/FEA job cancel。

**opencode 1.15.10 差异化研究**
和 1.3.13 对比：SDK data model 变更、tool calling 行为差异、MCP 协议支持程度、性能对比，最终给出升级策略结论。

### P2 · 下迭代

**opencode 升级执行**（前置：P1 研究结论）
CNB 构建镜像、Nacos namespace、dev 机部署新容器 + 数据卷隔离、切流测试。

**MCP 服务端主动通知（替代 polling）**
FEA 完成 → 回调 Gateway，不再让 opencode 轮询。MCP server completion callback → Gateway 路由到 SSE → 渐进式迁移。

**前端 SSE 断线重连 / 会话保持**
刷新不丢会话，断线自动恢复。SSE reconnection with Last-Event-ID + session 恢复推送 + 状态持久化。

### P3 · 技术债

- **Sidecar → Runtime Gateway 迁移**：BFF 去掉直连 sidecar，所有 runtime 调用经 Gateway 路由
- **deploy 分支同步**：7 个 repo 的 deploy 分支 rebase 到 develop
- **跨服务 tracing 统一**：request_id 贯穿 BFF → Gateway → MCP
