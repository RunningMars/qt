# QT - 币安合约量化交易系统

币安 USDT-M 永续合约全自动量化交易系统，基于 DeepSeek R1 大模型动态选标的与分析决策。

## 系统特点

- **全自动交易**：定时触发，无需人工干预，日线/周线低频策略
- **AI 驱动决策**：DeepSeek R1 综合技术面 + 新闻舆情，动态选择成交量 Top 20 标的
- **精简架构**：1+1 模式（数据服务 + 主服务），通过 MySQL + Redis 解耦，非微服务
- **严格风控**：杠杆上限、仓位限制、止损强平、日亏损熔断，代码硬编码不可覆盖
- **全 Go 技术栈**：统一语言，统一部署，问题定位简单

## 技术栈

| 层级 | 技术 |
|------|------|
| 后端 | Go |
| 前端 | Vue 3 |
| 数据库 | MySQL + Redis |
| 交易所 | Binance Futures API |
| AI 模型 | DeepSeek R1（OpenAI 兼容接口） |
| 部署 | Docker Compose |

## 架构概览

```
Vue 3 前端（监控看板）
       │
  主服务（Go）── 定时调度 → 指标计算 → R1分析 → 风控 → 下单
       │
  MySQL + Redis
       │
  数据服务（Go）── Binance WebSocket/REST + 新闻/舆情 API
```

## 文档

| 文档 | 说明 |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 系统架构设计：技术选型、模块划分、行业调研 |
| [IMPLEMENTATION.md](./IMPLEMENTATION.md) | 实现细节：API 封装、数据库设计、部署方案 |

## 开发环境

| 工具 | 版本 |
|------|------|
| Go | 1.22+ |
| MySQL | 8.0+ |
| Redis | 7.0+ |
| Docker | 24+ |
| Node.js | 18+（前端构建） |
