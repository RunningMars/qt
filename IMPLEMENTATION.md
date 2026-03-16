# 实现细节与开发指南

> v1.0 | 2026-03-16

本文档补充 [ARCHITECTURE.md](./ARCHITECTURE.md) 中的实现层面细节，指导开发人员落地具体功能。

---

## 1. Binance Futures API 封装

### 方案

基于 `adshao/go-binance/v2` 库，它已内置 HMAC-SHA256 签名和 Futures 完整支持。
如需定制化（如更细粒度的错误处理），可在其基础上封装一层内部 client。

### 核心接口

| 接口 | 用途 | API |
|------|------|-----|
| 成交量排行 | 获取 Top 20 标的 | `GET /fapi/v1/ticker/24hr` |
| 历史 K线 | 拉取日线/周线 | `GET /fapi/v1/klines` |
| 账户信息 | 余额和持仓 | `GET /fapi/v2/account` |
| 下单 | 开仓/平仓 | `POST /fapi/v1/order` |
| 修改杠杆 | 设置逐仓杠杆 | `POST /fapi/v1/leverage` |
| 修改保证金模式 | 切换逐仓 | `POST /fapi/v1/marginType` |
| 资金费率 | 当前费率 | `GET /fapi/v1/fundingRate` |
| WebSocket | 实时行情流 | `wss://fstream.binance.com/ws/` |

### HMAC 签名流程

```
1. 拼接请求参数为 query string
2. 追加 timestamp=当前毫秒时间戳
3. 用 Secret Key 对 query string 做 HMAC-SHA256
4. 将签名附加到请求参数：&signature=xxx
5. 请求头带 X-MBX-APIKEY
```

### 错误处理

- 网络错误：指数退避重试（最多 3 次）
- 限流（429）：等待 Retry-After 秒后重试
- 签名错误（-1022）：检查时钟同步
- 余额不足（-2019）：记录日志，跳过本次下单
- 下单被拒：记录完整请求和响应，便于排查

---

## 2. 数据服务实现

### WebSocket 多标的并发订阅

```
每个标的一个 goroutine：

goroutine 1: BTCUSDT kline_1d → 解析 → 写 Redis
goroutine 2: ETHUSDT kline_1d → 解析 → 写 Redis
...
goroutine 20: SOLUSDT kline_1d → 解析 → 写 Redis
```

Binance 支持 Combined Stream（单连接多标的），优先使用：
```
wss://fstream.binance.com/stream?streams=btcusdt@kline_1d/ethusdt@kline_1d/...
```

### 断线重连策略

```
检测断线（ping/pong 超时 30s 无响应）
    ↓
关闭旧连接
    ↓
指数退避等待（1s → 2s → 4s → 8s → 最大 60s）
    ↓
重新建立连接
    ↓
恢复全部订阅
    ↓
REST 回填重连期间遗漏的 K线数据
    ↓
重置退避计数器
```

### 数据写入

**MySQL（批量写入历史 K线）**：
- 每分钟批量 INSERT，而非逐条写入
- 使用 `INSERT ... ON DUPLICATE KEY UPDATE` 避免重复

**Redis（实时数据）**：
- Hash 结构：`realtime:{symbol}` → `{price, volume, change_pct, timestamp}`
- 恐贪指数：`fear_greed_index` → `{value, timestamp}`
- TTL：实时数据 5 分钟过期，防止脏数据

### 新闻/舆情采集

| 数据源 | 轮询间隔 | 存储 |
|--------|---------|------|
| CryptoPanic | 每 15 分钟 | MySQL（标题、情绪标签、时间） |
| Alternative.me | 每 1 小时 | Redis（恐贪指数值） |
| CoinGecko Trending | 每 1 小时 | Redis（热门标的列表） |

---

## 3. 主服务实现

### 定时任务编排（robfig/cron）

```
每日 UTC 00:05（币安日K收盘后 5 分钟）
    ↓
Step 1: 查询 Top 20 成交量标的     （超时 30s）
    ↓
Step 2: 批量拉取 K线 + 计算指标    （超时 60s）
    ↓
Step 3: 读取新闻 + 恐贪指数        （超时 10s）
    ↓
Step 4: 构建 Prompt → 调用 R1      （超时 120s）
    ↓
Step 5: 风控校验                   （超时 5s）
    ↓
Step 6: 执行下单                   （超时 30s）
    ↓
Step 7: 记录结果 → 推送前端         （超时 10s）
```

任何 Step 失败：记录日志 + 告警通知，不继续后续步骤。

### R1 Prompt 设计要点

- 使用 system prompt 定义输出 JSON Schema，确保结构化输出
- 每次调用前验证数据完整性（指标全部计算完成、新闻已拉取）
- R1 返回后做 JSON 解析，失败则重试一次（换 temperature）
- 置信度 < 60% 的信号不执行，记录到日志供复盘

### 风控实现

**强平距离计算**（逐仓模式）：

```
做多强平价 = 开仓价 × (1 - 1/杠杆 + 维持保证金率)
做空强平价 = 开仓价 × (1 + 1/杠杆 - 维持保证金率)
距强平距离 = |当前价 - 强平价| / 当前价
```

**日亏损熔断**：
- 每日 UTC 00:00 重置当日已实现盈亏计数器
- 每笔交易完成后累加盈亏
- 当日亏损 > 总资金 × 3% → 停止当日所有新开仓

### 订单状态机实现

```go
type OrderState int

const (
    OrderCreated        OrderState = iota
    OrderSubmitted
    OrderAccepted
    OrderWorking
    OrderPartiallyFilled
    OrderFilled         // 终态
    OrderCancelled      // 终态
)
```

状态转换通过 MySQL 记录，每次状态变更写入 orders 表。
异常恢复：主服务启动时检查 Working 状态的订单，向 Binance 查询最新状态并同步。

---

## 4. 数据库设计

### MySQL 表结构

**klines（K线历史）**
```sql
CREATE TABLE klines (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    symbol      VARCHAR(20)  NOT NULL,
    interval_   VARCHAR(5)   NOT NULL,  -- 1d, 1w
    open_time   BIGINT       NOT NULL,
    open        DECIMAL(20,8) NOT NULL,
    high        DECIMAL(20,8) NOT NULL,
    low         DECIMAL(20,8) NOT NULL,
    close       DECIMAL(20,8) NOT NULL,
    volume      DECIMAL(20,8) NOT NULL,
    close_time  BIGINT       NOT NULL,
    UNIQUE KEY uk_symbol_interval_time (symbol, interval_, open_time)
) ENGINE=InnoDB;
```

**news（新闻舆情）**
```sql
CREATE TABLE news (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    source      VARCHAR(30)  NOT NULL,  -- cryptopanic, coingecko
    title       VARCHAR(500) NOT NULL,
    sentiment   VARCHAR(20),            -- bullish, bearish, neutral
    symbols     VARCHAR(200),           -- 关联标的，逗号分隔
    published_at DATETIME    NOT NULL,
    created_at  DATETIME     DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_published (published_at),
    INDEX idx_symbols (symbols)
) ENGINE=InnoDB;
```

**trades（交易记录）**
```sql
CREATE TABLE trades (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    signal_id       BIGINT       NOT NULL,  -- 关联 R1 分析结果
    symbol          VARCHAR(20)  NOT NULL,
    direction       VARCHAR(10)  NOT NULL,  -- long, short
    leverage        INT          NOT NULL,
    entry_price     DECIMAL(20,8),
    exit_price      DECIMAL(20,8),
    quantity        DECIMAL(20,8) NOT NULL,
    pnl             DECIMAL(20,8),          -- 已实现盈亏
    status          VARCHAR(20)  NOT NULL,  -- open, closed
    opened_at       DATETIME     NOT NULL,
    closed_at       DATETIME,
    INDEX idx_status (status),
    INDEX idx_symbol (symbol)
) ENGINE=InnoDB;
```

**orders（订单记录）**
```sql
CREATE TABLE orders (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    trade_id        BIGINT       NOT NULL,
    exchange_order_id VARCHAR(50),
    symbol          VARCHAR(20)  NOT NULL,
    side            VARCHAR(10)  NOT NULL,  -- BUY, SELL
    type            VARCHAR(20)  NOT NULL,  -- MARKET, LIMIT, STOP_MARKET
    price           DECIMAL(20,8),
    quantity        DECIMAL(20,8) NOT NULL,
    filled_quantity DECIMAL(20,8) DEFAULT 0,
    status          VARCHAR(20)  NOT NULL,  -- created, submitted, working, filled, cancelled
    created_at      DATETIME     DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_trade (trade_id),
    INDEX idx_status (status)
) ENGINE=InnoDB;
```

**signals（R1 分析结果）**
```sql
CREATE TABLE signals (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    raw_prompt      TEXT         NOT NULL,  -- 发给 R1 的完整 Prompt
    raw_response    TEXT         NOT NULL,  -- R1 原始返回
    parsed_json     JSON,                   -- 解析后的 JSON
    confidence      VARCHAR(20),            -- high, medium, low
    executed        TINYINT      DEFAULT 0, -- 是否执行
    created_at      DATETIME     DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_created (created_at)
) ENGINE=InnoDB;
```

**funding_rates（资金费率）**
```sql
CREATE TABLE funding_rates (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    symbol          VARCHAR(20) NOT NULL,
    funding_rate    DECIMAL(10,6) NOT NULL,
    funding_time    BIGINT      NOT NULL,
    UNIQUE KEY uk_symbol_time (symbol, funding_time)
) ENGINE=InnoDB;
```

### Redis 数据结构

| Key | 类型 | 内容 | TTL |
|-----|------|------|-----|
| `realtime:{symbol}` | Hash | price, volume, change_pct, timestamp | 5min |
| `fear_greed_index` | String | JSON: {value, classification, timestamp} | 2h |
| `trending_coins` | List | 热门标的列表 | 2h |
| `daily_pnl` | String | 当日累计盈亏 | 至次日 00:00 |

---

## 5. 前端实现

### Vue 3 技术栈

- 构建工具：Vite
- UI 框架：待定（Element Plus or Ant Design Vue）
- 图表：TradingView Lightweight Charts（K线图）+ ECharts（盈亏曲线）

### 核心页面

| 页面 | 功能 |
|------|------|
| Dashboard | 账户总览、当日盈亏、持仓状态 |
| 持仓管理 | 当前持仓列表、强平距离、浮动盈亏 |
| 信号日志 | R1 分析历史、Prompt/Response 完整记录 |
| 交易历史 | 已完成交易列表、盈亏统计 |
| 系统状态 | 数据服务/主服务运行状态、WebSocket 连接状态 |

### WebSocket 推送

主服务通过 WebSocket 向前端推送：
- 持仓变动
- 新交易信号
- 订单状态更新
- 系统告警

---

## 6. 回测模块

### 基于 cinar/indicator 回测框架

cinar/indicator/v2 自带回测能力，核心流程：

```
加载历史 K线（MySQL）
    ↓
逐 K线回放：计算指标 → 模拟 R1 信号 → 风控校验 → 模拟下单
    ↓
统计结果：胜率、盈亏比、最大回撤、Sharpe 比率
```

### 建模参数

| 参数 | 值 | 说明 |
|------|----|------|
| 滑点 | 0.1~0.3% | 模拟实际成交偏差 |
| Taker 手续费 | 0.04% | 市价单 |
| Maker 手续费 | 0.02% | 限价单 |
| 资金费率 | 从 funding_rates 表读取实际历史值 | |

---

## 7. 部署方案

### Docker Compose

```yaml
services:
  data-service:
    build: ./data-service
    restart: always
    depends_on:
      - mysql
      - redis
    environment:
      - BINANCE_API_KEY=${BINANCE_API_KEY}
      - BINANCE_SECRET_KEY=${BINANCE_SECRET_KEY}

  main-service:
    build: ./main-service
    restart: always
    depends_on:
      - mysql
      - redis
    ports:
      - "8080:8080"
    environment:
      - BINANCE_API_KEY=${BINANCE_API_KEY}
      - BINANCE_SECRET_KEY=${BINANCE_SECRET_KEY}
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY}

  frontend:
    build: ./frontend
    ports:
      - "3000:80"

  mysql:
    image: mysql:8.0
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
```

### 环境变量

```
BINANCE_API_KEY=       # 币安 API Key
BINANCE_SECRET_KEY=    # 币安 Secret Key
DEEPSEEK_API_KEY=      # DeepSeek API Key
MYSQL_DSN=             # MySQL 连接串
REDIS_ADDR=            # Redis 地址
```

### 建议先使用 Binance 测试网

开发阶段使用 Binance Futures Testnet：
- REST: `https://testnet.binancefuture.com`
- WebSocket: `wss://stream.binancefuture.com`
- 免费测试资金，API 接口与生产环境一致
