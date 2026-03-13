# QT - 量化自动交易系统

基于事件驱动微服务架构的 AI 量化交易系统，支持多交易所、多策略并行运行。

## 核心特性

- **多 AI 模型协同**：DeepSeek R1 + Claude + 本地轻量模型
- **事件驱动架构**：Kafka 消息总线，服务解耦
- **实时流处理**：Flink 毫秒级技术指标计算
- **多层风控**：单笔/单日/账户三级熔断
- **全自动 & 半自动**：支持人工审批模式

## 文档

- [优化设计方案](./docs/DESIGN_OPTIMIZED.md) - 完整架构设计、技术栈、实施路线图

## 快速开始

> 项目正在建设中，详见 [实施路线图](./docs/DESIGN_OPTIMIZED.md#实施路线图)

## 技术栈

**后端**：Python 3.12 · FastAPI · Kafka · Flink · TimescaleDB · Redis  
**AI**：DeepSeek R1 · Claude API · PyTorch  
**前端**：React 18 · Vite · Ant Design · TradingView Charts  
**运维**：Docker · Kubernetes · Prometheus · Grafana



Init 1.0.0
