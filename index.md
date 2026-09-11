# 多智能体协作股票交易系统 — 实施方案

> 本文档供 AI 智能体阅读使用，完整描述系统架构、模块职责、接口定义与实现步骤。
> 其他 AI 可直接按此文档实现，无需额外上下文。
> 文件版本：2026-04-01

---

## 一、项目概述

**项目名称**：Multi-Agent Stock Trading System（多智能体协作股票交易系统）

**核心目标**：构建一个模块化的 A 股智能投研系统，通过多个专业化 AI Agent 协作，实现：
- 国家队资金追踪分析
- K 线技术图形识别
- 基本面多维评估
- 决策信号融合
- 风险闭环控制

**目标用户**：郑钰桦（北京师范大学，金融科技方向）

**技术约束**：
- Python 3.11 + Miniconda
- Windows 环境（开发机）
- SQLite 本地数据库（零配置）
- Kronos-mini 图模型（4.1M 参数，CPU 可跑，已下载于本地）
- 预算：500 元

---

## 二、系统架构

```
┌─────────────────────────────────────────────────────┐
│                   消息总线 (Message Bus)             │
│            agents 之间异步通信，事件驱动              │
└────┬──────┬──────┬──────┬──────┬──────┬─────────────┘
     │      │      │      │      │
     ▼      ▼      ▼      ▼      ▼
┌─────────┐┌──────┐┌──────┐┌─────────┐┌──────────┐
│国家队因子││技术分析││基本面  ││Kronos图 ││情绪分析   │
│Agent    ││Agent  ││Agent  ││Agent    ││(可选)    │
└────┬────┘└───┬──┘└───┬───┘└────┬────┘└────┬─────┘
     │         │        │         │           │
     └─────────┴───┬────┴─────────┘           │
                   ▼                           │
            ┌──────────┐                       │
            │决策融合   │                       │
            │Decision   │                       │
            │Agent      │                       │
            └─────┬─────┘                       │
                  │                             │
         ┌────────┴────────┐                   │
         ▼                  ▼                   │
  ┌──────────┐      ┌──────────────┐            │
  │交易执行   │      │风险控制      │            │
  │Agent     │      │Agent         │            │
  └─────┬────┘      └──────┬───────┘            │
        │                  │                    │
        ▼                  ▼                    │
  ┌──────────┐      ┌──────────────┐            │
  │ 回测引擎 │      │ SQLite数据库  │            │
  │Backtest  │◄────►│  (data.db)   │◄───────────┘
  └──────────┘      └──────────────┘
```

---

## 三、目录结构

```
multi_agent_stock/
├── README.md
├── IMPLEMENTATION.md          # 本文档（AI 实现指南）
├── ARCHITECTURE.md
├── requirements.txt
│
├── config/
│   ├── __init__.py
│   ├── settings.py           # 全局配置（股票池、参数、日志）
│   └── prompts.py            # 各 Agent 提示词模板
│
├── data/
│   ├── data.db               # SQLite 数据库文件
│   ├── minute_data/          # 分钟K线（CSV 分文件存储）
│   └── raw_data/             # 爬虫原始数据
│
├── core/                      # 核心基础设施
│   ├── __init__.py
│   ├── database.py           # 数据库操作层（CRUD、查询接口）
│   ├── data_fetcher.py       # 数据获取（baostock 封装）
│   ├── gov_holdings.py       # 国家队持仓爬虫（东方财富）
│   └── message_bus.py        # 智能体消息总线（事件驱动）
│
├── agents/                    # 所有智能体
│   ├── __init__.py
│   ├── base.py               # Agent 基类（抽象接口）
│   ├── agent_registry.py     # Agent 注册表（统一调度）
│   │
│   ├── analysis/             # ── 分析层 ──
│   │   ├── __init__.py
│   │   ├── gov_factor_agent.py      # 国家队因子分析
│   │   ├── tech_analysis_agent.py   # 技术分析
│   │   ├── fundamental_agent.py      # 基本面分析
│   │   ├── kronos_agent.py          # Kronos 图模型
│   │   └── emotion_agent.py         # 情绪分析（可选）
│   │
│   ├── decision/
│   │   ├── __init__.py
│   │   └── decision_agent.py        # 决策融合
│   │
│   └── execution/
│       ├── __init__.py
│       ├── trading_agent.py          # 交易执行
│       └── risk_control_agent.py     # 风险控制
│
├── models/                    # 模型相关
│   ├── __init__.py
│   ├── kronos/
│   │   ├── model/             # Kronos 模型文件（.pth/.onnx）
│   │   ├── predictor.py       # Kronos 预测封装
│   │   └── config.py          # 模型配置
│   │
│   └── indicators/
│       ├── __init__.py
│       └── technical.py       # 技术指标计算（纯 Python 实现）
│
├── backtest/                  # 回测模块
│   ├── __init__.py
│   ├── engine.py              # 事件驱动回测引擎
│   ├── metrics.py             # 绩效指标（夏普/回撤/胜率等）
│   └── visualizer.py          # 结果可视化（matplotlib）
│
├── utils/
│   ├── __init__.py
│   ├── logger.py              # 日志配置
│   └── helpers.py            # 工具函数
│
├── scripts/
│   ├── install_env.sh         # 环境安装脚本
│   ├── init_db.py             # 初始化数据库
│   └── run_demo.py            # 运行 Demo
│
└── tests/
    ├── __init__.py
    ├── test_agents.py
    ├── test_data.py
    └── test_kronos.py
```

---

## 四、数据库设计

### 4.1 表结构（SQLite）

```sql
-- 日线行情（由 baostock 填充）
CREATE TABLE daily_price (
    code        TEXT,          -- 股票代码 e.g. '600519.SH'
    date        TEXT,          -- 日期 e.g. '2026-03-31'
    open        REAL,
    high        REAL,
    low         REAL,
    close       REAL,
    volume      REAL,
    amount      REAL,
    PRIMARY KEY (code, date)
);

-- 分钟K线（可选，由东方财富爬虫填充）
CREATE TABLE minute_price (
    code        TEXT,
    datetime    TEXT,          -- e.g. '2026-03-31 09:30:00'
    open        REAL,
    high        REAL,
    low         REAL,
    close       REAL,
    volume      REAL,
    PRIMARY KEY (code, datetime)
);

-- 国家队持仓（东方财富爬虫）
CREATE TABLE gov_holdings (
    code        TEXT,          -- 股票代码
    holder_name TEXT,          -- 持仓机构名（证金/汇金/社保）
    quarter     TEXT,          -- 季度 e.g. '2025Q4'
    shares      REAL,          -- 持股数量（万股）
    ratio       REAL,          -- 占总股本比例
    change      REAL,          -- 较上期变化
    updated_at  TEXT,
    PRIMARY KEY (code, holder_name, quarter)
);

-- 财务报表
CREATE TABLE financial_statements (
    code        TEXT,
    date        TEXT,
    type        TEXT,          -- 'bs'(资产负债表) / 'is'(利润表) / 'cf'(现金流量表)
    items       TEXT,          -- JSON 格式存储所有科目
    PRIMARY KEY (code, date, type)
);

-- Agent 分析结果缓存（避免重复计算）
CREATE TABLE agent_results (
    code        TEXT,
    agent_name  TEXT,           -- e.g. 'gov_factor', 'tech', 'fundamental'
    signal      TEXT,          -- 'buy' / 'hold' / 'sell'
    score       REAL,          -- 0~100
    detail      TEXT,          -- JSON 详细输出
    computed_at TEXT,
    PRIMARY KEY (code, agent_name)
);

-- 交易记录
CREATE TABLE trades (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    code        TEXT,
    date        TEXT,
    action      TEXT,          -- 'buy' / 'sell'
    price       REAL,
    shares      INTEGER,
    commission  REAL,
    signal_src  TEXT,          -- 触发来源
    created_at  TEXT
);

-- 回测结果
CREATE TABLE backtest_results (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    strategy    TEXT,
    start_date  TEXT,
    end_date    TEXT,
    total_return REAL,
    sharpe_ratio REAL,
    max_drawdown REAL,
    win_rate    REAL,
    params      TEXT,          -- JSON
    created_at  TEXT
);
```

---

## 五、核心模块实现规范

### 5.1 配置层 `config/settings.py`

```python
# 全局配置（须在 settings.py 中定义）
STOCK_POOL = ['600519.SH', '000858.SZ', '601318.SH']  # 默认股票池
DATA_DIR = 'data'
DB_PATH = 'data/data.db'

# 数据源
BAOSTOCK_TOKEN = None  # baostock 免费，无需 token

# Agent 配置
AGENTS = {
    'gov_factor':   {'enabled': True,  'weight': 0.25},
    'tech':          {'enabled': True,  'weight': 0.30},
    'fundamental':   {'enabled': True,  'weight': 25},
    'kronos':        {'enabled': True,  'weight': 0.20},
}
DECISION_THRESHOLD = 60  # 买入阈值（综合评分 > 60）
RISK_MAX_POSITION = 0.2  # 单只最大仓位 20%

# Kronos 模型路径
KRONOS_MODEL_DIR = 'models/kronos/model'
KRONOS_MODEL_FILE = 'kronos_mini.pth'  # 本地已有

# 回测参数
BACKTEST_INITIAL_CASH = 1000000  # 初始资金 100 万
BACKTEST_COMMISSION = 0.0003     # 万三手续费
BACKTEST_SLIPPAGE = 0.001        # 千一滑点

# 日志
LOG_LEVEL = 'INFO'
LOG_FILE = 'logs/agent.log'
```

### 5.2 数据库层 `core/database.py`

**必须实现**：
- `init_db()` — 建表（读建表 SQL 执行）
- `save_daily_price(code, date, OHLCV)` — 存日线
- `get_daily_price(code, start, end)` → DataFrame
- `save_gov_holdings(records: list[dict])` — 存国家队数据
- `get_gov_holdings(code)` → DataFrame
- `save_agent_result(code, agent, signal, score, detail)` — 缓存分析结果
- `get_agent_result(code, agent)` — 读缓存
- `save_trade(...)` / `get_trades(code, start, end)`
- `save_backtest_result(...)` / `get_backtest_results()`

**技术要求**：
- 使用 `sqlite3` 标准库，不引入额外依赖
- 所有写入使用 `contextlib.contextmanager` 管理连接
- 查询结果统一返回 `pandas.DataFrame`

### 5.3 数据获取层 `core/data_fetcher.py`

**baostock 封装接口**：

```python
def fetch_daily(code: str, start: str, end: str) -> pd.DataFrame:
    """获取日线数据，返回 columns=[date,open,high,low,close,volume,amount]"""

def fetch_financial(code: str, year: int, quarter: int) -> dict:
    """获取季报/年报，返回 items dict"""
```

**国家队爬虫 `core/gov_holdings.py`**：

```python
def crawl_gov_holdings(quarter: str = '2025Q4') -> list[dict]:
    """从东方财富爬取国家队持仓数据"""
    # URL: https://datacenter-web.eastmoney.com/api/data/v1/get
    # 参数: reportName=RPT_HOLDERGATHER_STA&columns=ALL
    # 解析: 汇金(中金/汇金资管)、证金、社保持仓
    # 返回: [{code, holder_name, quarter, shares, ratio, change}, ...]
```

### 5.4 消息总线 `core/message_bus.py`

**事件驱动通信**：

```python
class MessageBus:
    # Agent 订阅主题，发布时自动调度
    def subscribe(agent_name: str, topic: str, callback: Callable)
    def publish(topic: str, payload: dict)  # 广播给所有订阅者
    def send(from_agent: str, to_agent: str, payload: dict)  # 点对点
```

**事件类型**：
- `data.ready` — 数据就绪（触发所有分析 Agent）
- `analysis.done.{agent}` — 单个 Agent 分析完成
- `decision.ready` — 决策完成
- `trade.signal` — 交易信号
- `risk.alert` — 风险告警

---

## 六、智能体实现规范

### 6.1 Agent 基类 `agents/base.py`

```python
from abc import ABC, abstractmethod
from typing import Any

class BaseAgent(ABC):
    name: str           # 全局唯一标识
    weight: float       # 决策权重 0~1

    @abstractmethod
    def analyze(self, context: dict) -> dict:
        """
        输入: context = {
            'code': str,
            'start_date': str,
            'end_date': str,
            'daily_data': DataFrame,
            'gov_data': DataFrame | None,
            'financial_data': dict | None,
        }
        输出: {
            'signal': 'buy' | 'hold' | 'sell',
            'score': float,       # 0~100
            'confidence': float,  # 置信度 0~1
            'reason': str,         # 简短原因
            'detail': dict         # 详细数据（用于决策融合）
        }
        """

    def reset(self):
        """重置状态（可选）"""
        pass
```

### 6.2 国家队因子 Agent `agents/analysis/gov_factor_agent.py`

**分析逻辑**：

1. 从数据库读取该股票近4个季度的国家队持仓
2. 计算 Z-score：`z = (当前占比 - 均值) / 标准差`
3. 计算斜率：`slope = 线性回归斜率(季度序列)`
4. 综合因子：`factor = z * 0.6 + slope * 0.4`
5. 映射到信号：
   - factor > 0.5 → **buy** (score = 60 + factor*40)
   - -0.3 < factor <= 0.5 → **hold** (score = 40 + factor*30)
   - factor <= -0.3 → **sell** (score = 20 + factor*20)

**输出 detail 字段**：
```python
{
    'z_score': float,
    'slope': float,
    'factor': float,
    'holders': [{'name': str, 'ratio': float, 'change': float}, ...],
    'quarterly_ratios': [list of float]
}
```

### 6.3 技术分析 Agent `agents/analysis/tech_analysis_agent.py`

**纯 Python 实现指标计算**（`models/indicators/technical.py`），不使用 Talib：

```python
# 必须实现的指标（均为纯 Python + numpy）
def calc_ma(close: Series, n: int) -> Series          # 简单均线
def calc_ema(close: Series, n: int) -> Series         # 指数均线
def calc_macd(close: Series, fast=12, slow=26, signal=9) -> dict
def calc_rsi(close: Series, n=14) -> Series
def calc_kdj(high, low, close, n=9) -> dict             # K/D/J
def calc_boll(close: Series, n=20, k=2) -> dict        # 布林带
```

**买卖信号生成**：
```python
signals = {
    'MA_cross':      'gold_cross' | 'death_cross' | 'none',
    'MACD_cross':    'above' | 'below' | 'none',
    'RSI_level':     'overbought' | 'oversold' | 'neutral',
    'KDJ_position': 'overbought' | 'oversold' | 'neutral',
    'BOLL_position': 'upper突破' | 'lower突破' | 'band内'
}

# 综合评分逻辑（权重可配置）：
score = w1*MA_score + w2*MACD_score + w3*RSI_score + w4*KDJ_score + w5*BOLL_score
```

### 6.4 Kronos 图模型 Agent `agents/analysis/kronos_agent.py`

**Kronos-mini 使用规范**：

Kronos-mini 是时间序列预测模型，输入为时间序列特征图（多变量），输出预测走势。

**predictor.py 封装**：
```python
import torch
import sys
sys.path.insert(0, 'models/kronos')

class KronosPredictor:
    def __init__(self, model_path: str, device='cpu'):
        # 加载本地 .pth 模型文件
        # 本地路径: C:\Users\Administrator\.qclaw\workspace\Kronos\Kronos\model\

    def predict(self, df: DataFrame, lookback=60, horizon=5) -> dict:
        """
        df: 日线数据（至少 lookback 条）
        返回: {
            'forecast': [float, ...],     # 未来 horizon 日预测收盘价
            'trend': 'up' | 'down' | 'sideways',
            'confidence': float 0~1,
            'raw_output': tensor           # 原始模型输出（调试用）
        }
        """
```

**Agent 输出**：
- 预测趋势为 up → signal=buy, score = 70 + confidence*30
- 预测趋势为 down → signal=sell, score = 30 - confidence*30
- 预测趋势为 sideways → signal=hold, score=50

### 6.5 基本面 Agent `agents/analysis/fundamental_agent.py`

**分析维度**：

```python
# 1. 估值指标
valuation = {
    'PE': current_pe,      # 市盈率
    'PB': current_pb,      # 市净率
    'ROE': net_profit / equity,  # 净资产收益率
    'EPS': earnings / shares,
}

# 2. 财务健康度
health = {
    'current_ratio': current_assets / current_liabilities,
    'debt_ratio': total_liabilities / total_assets,
    'profit_growth': (current_profit - last_profit) / last_profit,
}

# 3. 综合评分
fundamental_score = weighted_sum(valuation, health)

# 4. 行业对比（获取同业股票数据做 percentile 排名）
```

**输出**：
- score > 70 → buy
- 40 < score <= 70 → hold
- score <= 40 → sell

### 6.6 决策融合 Agent `agents/decision/decision_agent.py`

**融合逻辑**（动态加权平均）：

```python
def fuse(agent_results: list[dict]) -> dict:
    # agent_results: 各分析 Agent 的输出列表
    total_weight = sum(a['weight'] for a in agent_results if a['signal'] != 'hold')
    
    # buy/sell 方向加权
    buy_score = sum(a['score'] * a['weight'] for a in agent_results if a['signal'] == 'buy')
    sell_score = sum(a['score'] * a['weight'] for a in agent_results if a['signal'] == 'sell')
    hold_score = sum(a['score'] * a['weight'] for a in agent_results if a['signal'] == 'hold')
    
    net_score = (buy_score - sell_score) / total_weight if total_weight > 0 else 0
    
    if net_score > 20:
        final_signal = 'buy'
        final_score = min(100, 50 + net_score)
    elif net_score < -20:
        final_signal = 'sell'
        final_score = max(0, 50 + net_score)
    else:
        final_signal = 'hold'
        final_score = 50 + net_score * 0.5
    
    return {
        'signal': final_signal,
        'score': final_score,
        'buy_score': buy_score,
        'sell_score': sell_score,
        'contributors': [a['name'] for a in agent_results if a['signal'] == 'buy']
    }
```

### 6.7 交易执行 Agent `agents/execution/trading_agent.py`

```python
def execute_trade(signal: dict, position: dict, price: float):
    """
    signal: {'code', 'action': 'buy'/'sell', 'score', 'reason'}
    position: {'cash', 'holdings': {code: shares}}
    price: 当前价格（回测用收盘价）
    """
    # 计算买入数量（按风控 Agent 的 max_position 限制）
    # 生成订单记录
    # 对接回测引擎（虚拟撮合）
```

### 6.8 风险控制 Agent `agents/execution/risk_control_agent.py`

```python
# 风控规则：
MAX_DRAWDOWN = 0.15          # 最大回撤 15% 止损
MAX_SINGLE_POSITION = 0.20   # 单只仓位 <= 20%
MAX_DAILY_LOSS = 0.05        # 单日亏损 > 5% 停止交易

# VaR 计算（历史模拟法，95% 置信）
def calc_var(returns: Series, confidence=0.95) -> float:
    return np.percentile(returns, (1 - confidence) * 100)

# 持仓监控
def check_positions(portfolio: dict) -> dict:
    # 返回: {'alerts': [...], 'actions': [...]}
    # 超仓 → 强制减仓
    # 回撤超限 → 清仓止损
```

---

## 七、回测引擎 `backtest/engine.py`

**事件驱动回测框架**：

```python
class BacktestEngine:
    def __init__(self, initial_cash=1_000_000, commission=0.0003, slippage=0.001):
        self.cash = initial_cash
        self.positions = {}    # {code: shares}
        self.trades = []
        self.portfolio_values = []

    def run(self, start_date: str, end_date: str, agents: list[BaseAgent]):
        """
        1. 按日期遍历 start→end
        2. 每日：fetch数据 → notify(analysis) → notify(decision) → 执行交易
        3. 撮合：买入→按当日收盘价×(1+滑点)成交，卖出同理
        4. 记录每日净值
        """

    def match_buy(self, code: str, date: str, target_shares: int) -> bool:
        price = get_close(code, date) * (1 + self.slippage)
        cost = price * target_shares * (1 + self.commission)
        if cost <= self.cash:
            self.cash -= cost
            self.positions[code] = self.positions.get(code, 0) + target_shares
            return True
        return False

    def get_metrics(self) -> dict:
        # 计算：总收益、夏普比率、最大回撤、胜率、盈亏比、卡玛比率
```

**绩效指标 `backtest/metrics.py`**：
```python
def calc_metrics(portfolio_values: list[float], trades: list[dict]) -> dict:
    # total_return = (final - initial) / initial
    # sharpe_ratio = mean(daily_returns) / std(daily_returns) * sqrt(252)
    # max_drawdown = max(peak - current) / peak
    # win_rate = wins / (wins + losses)
    # profit_loss_ratio = avg_win / abs(avg_loss)
    # calmar_ratio = total_return / max_drawdown
```

---

## 八、依赖 `requirements.txt`

```
# 核心
pandas>=2.0.0
numpy>=1.24.0
sqlite3  # 标准库，无需安装

# 数据获取
baostock>=0.8.8
requests>=2.31.0

# 深度学习（Kronos CPU 推理）
torch>=2.0.0
numpy>=1.24.0

# 可视化
matplotlib>=3.7.0

# 工具
python-dateutil>=2.8.0
```

---

## 九、实现顺序（AI 执行顺序）

**第 1 步：搭建骨架**
- 创建目录结构
- 编写 `requirements.txt`
- 编写 `config/settings.py`（全局配置）
- 编写 `utils/logger.py`（日志）

**第 2 步：数据层**
- 编写 `core/database.py`（建表 + CRUD）
- 编写 `core/data_fetcher.py`（baostock 封装）
- 编写 `core/gov_holdings.py`（国家队爬虫）
- 执行 `scripts/init_db.py` 初始化数据库

**第 3 步：基础设施**
- 编写 `core/message_bus.py`（事件总线）
- 编写 `agents/base.py`（Agent 基类）
- 编写 `agents/agent_registry.py`（注册表）

**第 4 步：分析层 Agent（按此顺序）**
- `models/indicators/technical.py`（技术指标，纯 Python）
- `agents/analysis/tech_analysis_agent.py`（技术分析 Agent）
- `agents/analysis/gov_factor_agent.py`（国家队因子 Agent）
- `agents/analysis/fundamental_agent.py`（基本面 Agent）
- `models/kronos/predictor.py`（Kronos 封装）
- `agents/analysis/kronos_agent.py`（Kronos 图模型 Agent）

**第 5 步：决策与执行层**
- `agents/decision/decision_agent.py`（决策融合）
- `agents/execution/risk_control_agent.py`（风控）
- `agents/execution/trading_agent.py`（交易执行）

**第 6 步：回测系统**
- `backtest/engine.py`（回测引擎）
- `backtest/metrics.py`（绩效指标）
- `backtest/visualizer.py`（可视化）

**第 7 步：主程序 + Demo**
- `main.py`（命令行入口）
- `scripts/run_demo.py`（快速演示脚本）

**第 8 步：文档**
- `README.md`（使用说明）
- `ARCHITECTURE.md`（架构文档）

---

## 十、Kronos 模型加载关键点

- **模型文件路径**：`C:\Users\Administrator\.qclaw\workspace\Kronos\Kronos\model\`
- **模型格式**：`.pth`（PyTorch）
- **运行设备**：`torch.device('cpu')`（无需 GPU）
- **输入格式**：需要向量化后的时间序列特征（OHLCV + 指标）
- **lookback window**：建议 60 天
- **预测 horizon**：5 天
- **加载示例**：
```python
import torch
model = torch.load(model_path, map_location='cpu')
model.eval()
with torch.no_grad():
    output = model(input_tensor)
```

---

## 十一、Demo 运行流程

```bash
# 1. 创建环境
conda create -n stock-agent python=3.11 -y
conda activate stock-agent
pip install -r requirements.txt

# 2. 初始化数据库
python scripts/init_db.py

# 3. 获取示例数据（贵州茅台 600519.SH，2024全年）
python -c "
from core.data_fetcher import fetch_daily
from core.database import save_daily_price
df = fetch_daily('600519.SH', '2024-01-01', '2024-12-31')
for _, row in df.iterrows():
    save_daily_price(row['code'], row['date'], row)
print('数据就绪')
"

# 4. 运行 Demo
python scripts/run_demo.py
```

**Demo 输出内容**：
- 贵州茅台各 Agent 分析结果（国家队/技术/基本面/Kronos）
- 决策融合信号
- 回测绩效报告（收益率/夏普/回撤）
- 资金曲线 PNG 图

---

## 十二、注意事项（AI 容易出错的地方）

1. **baostock 限制**：免费接口每分钟最多请求 200 次，每次最多 80 条记录，大批量数据需加 `sleep(0.01)`
2. **Kronos 输入格式**：必须严格按模型要求的 shape 构造 input tensor，否则输出无意义
3. **国家队数据时效性**：东方财富爬虫可能因反爬失效，须加入 `requests` 的 `headers` 和重试逻辑
4. **SQLite 并发**：`sqlite3` 不支持多线程写入，同一连接不要跨线程使用
5. **回测滑点**：滑点对高频策略影响极大，本项目按 0.1% 固定滑点保守估算
6. **Windows 路径**：所有文件路径用 `pathlib.Path` 或正斜杠，勿硬编码反斜杠
7. **中文编码**：Windows 下涉及文件读写或数据库字段时，统一用 UTF-8，避免乱码
8. **最小数据量**：Kronos 至少需要 60 条历史数据，技术指标至少需要 60 条（RSI/KDJ）

---

*本文档由 AI 辅助生成，供其他 AI 智能体实现使用。*
*如需补充任何模块的详细实现代码，请单独提出。*
