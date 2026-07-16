# Student Affairs Agent

面向高校学生事务的智能分诊与服务协同系统。

系统以真实高校独立部署为目标，使用 AI Agent、RAG 和持久化工作流实现政策咨询、信息补全、材料核验、事项分诊、人工审批、跨部门协同、结果回访与经验沉淀。

## Project Status

当前处于架构设计阶段。仓库尚未开始业务代码实现。

已确认的核心原则：

- 一所高校一个独立部署实例，可复制交付。
- 学生端和经办人端使用独立产品入口。
- `ServiceCase` 是唯一业务事实来源，聊天和 Agent Checkpoint 不能取代正式事项状态。
- LangGraph 负责编排、中断和恢复，业务状态转换必须经过确定性规则与授权命令处理器。
- PostgreSQL 保存业务事实、审计、Outbox 和 LangGraph Checkpoint。
- Milvus 提供 BGE-M3 稠密/稀疏混合检索，MinIO 保存政策与材料文件。
- 本地使用 Conda 和 Docker Compose；生产架构支持校内私有环境中的高可用与水平扩展。
- 本地环境可以缩小部署规模，但不能改变生产代码的职责边界和数据语义。

完整设计见 [生产级系统设计](docs/superpowers/specs/2026-07-16-student-affairs-platform-design.md)。

## Repository

- Local: `D:\agent\student-affairs-agent`
- GitHub: `https://github.com/NothingToSay0503/student-affairs-agent`
- Python package: `student_affairs`
