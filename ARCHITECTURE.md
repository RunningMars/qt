# 量化自动交易系统 - 顶层设计方案

## Context（背景）

本项目旨在构建一个自动化量化策略交易平台，专注于加密货币杠杆合约交易（做多/做空）。系统需要整合 OpenClaw、Claude Code 和 DeepSeek R1 模型 API 等 AI 能力，实现实时市场分析、自主决策、自动交易和风险控制。支持币安和欧易交易平台，提供全自动和半自动（人工审批）两种交易模式。

## 市场研究要点

### 量化交易系统核心要素
1. **数据层**：实时行情、历史数据、链上数据、市场情绪
2. **分析层**：技术指标、AI 模型预测、多因子分析
3. **策略层**：信号生成、仓位管理、风险控制
4. **执行层**：订单路由、滑点控制、执行监控
5. **监控层**：性能追踪、风险预警、回测验证

### 关键技术挑战
- 低延迟数据处理（WebSocket 实时流）
- 高频交易的并发控制
- 多交易所 API 统一抽象
- AI 模型推理速度与准确性平衡
- 资金安全与风险隔离

## 系统架构设计

### 技术栈选择
- **后端语言**：Python 3.11+（量化生态成熟，AI 集成便利）
- **异步框架**：asyncio + aiohttp（高并发 WebSocket 连接）
- **数据存储**：
  - TimescaleDB（时序数据）
  - Redis（实时缓存、消息队列）
  - PostgreSQL（策略配置、交易记录）
- **AI 集成**：
  - DeepSeek R1 API（策略决策）
  - Claude API（风险评估、策略优化）
- **监控**：Prometheus + Grafana
- **回测引擎**：Backtrader / VectorBT

### 目录结构
```
qt/
├── docs/                      # 文档
│   ├── architecture.md        # 架构设计
│   ├── api-integration.md     # API 对接文档
│   └── strategy-guide.md      # 策略开发指南
├── src/
│   ├── core/                  # 核心模块
│   │   ├── engine.py          # 交易引擎主控
│   │   ├── event_bus.py       # 事件总线
│   │   └── state_manager.py   # 状态管理
│   ├── data/                  # 数据层
│   │   ├── collectors/        # 数据采集器
│   │   │   ├── binance.py
│   │   │   └── okx.py
│   │   ├── processors/        # 数据处理
│   │   │   ├── indicators.py  # 技术指标
│   │   │   └── features.py    # 特征工程
│   │   └── storage/           # 数据存储
│   │       ├── timeseries.py
│   │       └── cache.py
│   ├── analysis/              # 分析层
│   │   ├── ai_models/         # AI 模型
│   │   │   ├── deepseek.py    # DeepSeek R1 集成
│   │   │   └── claude.py      # Claude 集成
│   │   ├── technical.py       # 技术分析
│   │   └── sentiment.py       # 市场情绪分析
│   ├── strategy/              # 策略层
│   │   ├── base.py            # 策略基类
│   │   ├── signals.py         # 信号生成
│   │   ├── position.py        # 仓位管理
│   │   └── risk.py            # 风险控制
│   ├── execution/             # 执行层
│   │   ├── broker.py          # 交易所抽象
│   │   ├── order_manager.py   # 订单管理
│   │   └── executor.py        # 执行器
│   ├── skills/                # Claude Code Skills
│   │   ├── strategy_optimizer.py  # 策略优化
│   │   ├── backtest_runner.py     # 回测执行
│   │   └── risk_analyzer.py       # 风险分析
│   └── monitoring/            # 监控层
│       ├── metrics.py         # 性能指标
│       ├── alerts.py          # 告警系统
│       └── dashboard.py       # 监控面板
├── strategies/                # 策略实现
│   ├── trend_following.py
│   ├── mean_reversion.py
│   └── ai_hybrid.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── backtest/
├── config/
│   ├── exchanges.yaml         # 交易所配置
│   ├── strategies.yaml        # 策略参数
│   └── risk_limits.yaml       # 风险限制
├── scripts/
│   ├── setup_db.py
│   └── migrate.py
├── requirements.txt
├── pyproject.toml
└── README.md
```

## 核心模块设计

### 1. 数据采集模块（data/collectors）
**功能**：
- WebSocket 实时订阅（K线、深度、成交）
- REST API 历史数据拉取
- 多交易所统一数据格式
- 断线重连与数据补全

**关键类**：
- `BaseCollector`：抽象基类
- `BinanceCollector`：币安实现
- `OKXCollector`：欧易实现

### 2. AI 决策模块（analysis/ai_models）
**功能**：
- DeepSeek R1 API 调用（策略推理）
- Claude API 调用（风险评估）
- 提示词工程与上下文管理
- 模型响应解析与验证

**集成方式**：
```python
# DeepSeek R1 用于快速决策
decision = await deepseek_client.analyze(
    market_data=current_state,
    strategy_context=strategy_params
)

# Claude 用于风险审查
risk_assessment = await claude_client.evaluate_risk(
    proposed_trade=decision,
    portfolio_state=portfolio
)
```

### 3. 策略引擎（strategy）
**功能**：
- 策略生命周期管理
- 信号生成与过滤
- 仓位计算（凯利公式、固定比例）
- 止盈止损逻辑

**策略基类接口**：
```python
class BaseStrategy:
    async def on_bar(self, bar: Bar) -> Signal
    async def on_tick(self, tick: Tick) -> Signal
    def calculate_position_size(self, signal: Signal) -> float
    def check_stop_loss(self, position: Position) -> bool
    def check_take_profit(self, position: Position) -> bool
```

### 4. 执行引擎（execution）
**功能**：
- 订单生命周期管理
- 滑点控制与订单拆分
- 执行反馈与状态同步
- 异常处理与重试机制

**执行模式**：
- **全自动模式**：信号直接执行
- **半自动模式**：信号发送至审批队列，人工确认后执行

### 5. 风险控制模块（strategy/risk）
**功能**：
- 单笔交易风险限制
- 账户总风险敞口控制
- 最大回撤监控
- 强制平仓机制

**风控规则**：
```yaml
risk_limits:
  max_position_size: 0.1          # 单仓位最大占比 10%
  max_leverage: 5                 # 最大杠杆倍数
  max_daily_loss: 0.05            # 单日最大亏损 5%
  max_drawdown: 0.15              # 最大回撤 15%
  stop_loss_pct: 0.02             # 止损比例 2%
  take_profit_pct: 0.05           # 止盈比例 5%
```

### 6. 回测系统（tests/backtest）
**功能**：
- 历史数据回放
- 策略性能评估（夏普比率、最大回撤）
- 参数优化（网格搜索、贝叶斯优化）
- 回测报告生成

## OpenClaw 集成方案

OpenClaw 作为 AI 代理框架，可用于：

1. **策略自动优化**：
   - 使用 Claude Code Skills 定期分析策略表现
   - 自动调整参数或建议新策略

2. **异常诊断**：
   - 监控系统日志，自动识别异常模式
   - 生成诊断报告并建议修复方案

3. **代码迭代**：
   - 基于回测结果自动重构策略代码
   - 生成单元测试

**实现方式**：
- 在 `src/skills/` 目录创建 Claude Code Skills
- 通过 MCP 协议与交易系统通信
- 定期触发优化任务（cron 或事件驱动）

## 交易所 API 对接

### 币安（Binance）
- **现货/合约 API**：`python-binance` 库
- **WebSocket**：实时 K线、深度、成交
- **认证**：API Key + Secret（存储在环境变量）

### 欧易（OKX）
- **统一 API**：`ccxt` 库（支持多交易所）
- **WebSocket**：订阅频道统一封装
- **认证**：API Key + Secret + Passphrase

**统一抽象层**：
```python
class ExchangeAdapter(ABC):
    @abstractmethod
    async def get_balance(self) -> Dict
    @abstractmethod
    async def place_order(self, order: Order) -> OrderResult
    @abstractmethod
    async def cancel_order(self, order_id: str) -> bool
    @abstractmethod
    async def get_positions(self) -> List[Position]
```

## 部署与运维

### 开发环境
- Docker Compose（数据库 + Redis + 应用）
- 模拟交易模式（使用测试网 API）

### 生产环境
- Kubernetes 部署（高可用）
- 密钥管理：Vault 或 K8s Secrets
- 日志聚合：ELK Stack
- 监控告警：Prometheus + AlertManager

## 实施路线图

### Phase 1：基础设施（2-3 周）
- [ ] 项目脚手架搭建
- [ ] 数据库设计与初始化
- [ ] 交易所 API 封装
- [ ] 实时数据采集器

### Phase 2：核心引擎（3-4 周）
- [ ] 事件驱动架构实现
- [ ] 策略基类与示例策略
- [ ] 订单执行引擎
- [ ] 风险控制模块

### Phase 3：AI 集成（2-3 周）
- [ ] DeepSeek R1 API 集成
- [ ] Claude API 集成
- [ ] AI 决策流程设计
- [ ] 提示词工程优化

### Phase 4：回测与优化（2-3 周）
- [ ] 回测引擎开发
- [ ] 性能指标计算
- [ ] 参数优化工具
- [ ] Claude Code Skills 开发

### Phase 5：监控与部署（1-2 周）
- [ ] 监控面板开发
- [ ] 告警系统配置
- [ ] Docker 镜像构建
- [ ] 生产环境部署

## 关键文件清单

### 配置文件
- `config/exchanges.yaml`：交易所 API 配置
- `config/strategies.yaml`：策略参数配置
- `config/risk_limits.yaml`：风险控制规则
- `.env`：敏感信息（API 密钥）

### 核心代码
- `src/core/engine.py`：交易引擎主控逻辑
- `src/data/collectors/binance.py`：币安数据采集
- `src/data/collectors/okx.py`：欧易数据采集
- `src/analysis/ai_models/deepseek.py`：DeepSeek 集成
- `src/analysis/ai_models/claude.py`：Claude 集成
- `src/strategy/base.py`：策略基类
- `src/execution/broker.py`：交易所抽象层
- `src/execution/order_manager.py`：订单管理器
- `src/strategy/risk.py`：风险控制器

### 策略示例
- `strategies/trend_following.py`：趋势跟踪策略
- `strategies/ai_hybrid.py`：AI 混合策略

### Skills
- `src/skills/strategy_optimizer.py`：策略优化 Skill
- `src/skills/backtest_runner.py`：回测执行 Skill

## 验证计划

### 单元测试
- 数据采集器测试（模拟 WebSocket）
- 策略逻辑测试（固定输入输出）
- 风控规则测试（边界条件）

### 集成测试
- 端到端交易流程（使用测试网）
- AI 模型调用测试
- 数据库读写测试

### 回测验证
- 使用历史数据验证策略有效性
- 对比不同参数配置的表现
- 压力测试（极端市场条件）

### 实盘测试
- 小资金量试运行（1-2 周）
- 监控执行质量与滑点
- 验证风控机制有效性

## 风险提示

1. **技术风险**：API 限流、网络延迟、系统故障
2. **市场风险**：极端行情、流动性不足、黑天鹅事件
3. **模型风险**：AI 决策失误、过拟合、数据偏差
4. **安全风险**：API 密钥泄露、账户被盗、恶意攻击

**缓解措施**：
- 多层风控机制
- 资金分散管理
- 实时监控告警
- 定期安全审计
