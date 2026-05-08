# A股自选股智能分析系统 - Code Wiki

## 目录

1. [项目概述](#1-项目概述)
2. [项目架构](#2-项目架构)
3. [主要模块职责](#3-主要模块职责)
4. [关键类与函数说明](#4-关键类与函数说明)
5. [依赖关系](#5-依赖关系)
6. [项目运行方式](#6-项目运行方式)
7. [配置说明](#7-配置说明)
8. [数据库模型](#8-数据库模型)

---

## 1. 项目概述

### 1.1 项目简介

**A股自选股智能分析系统** (Stock Analysis System) 是一款基于 AI 的智能股票分析工具，提供：

- **技术分析**：均线系统、K线形态、趋势识别、筹码分布
- **情报搜索**：新闻资讯、市场情绪、公告预警
- **风控评估**：多维度风险检测与信号降级
- **组合管理**：持仓跟踪、盈亏计算、FX汇率管理
- **多渠道通知**：微信、飞书、Telegram、邮件等

### 1.2 核心特性

| 特性 | 描述 |
|------|------|
| 多Agent架构 | Technical/Intel/Risk/Specialist/Decision 流水线 |
| 多数据源 | efinance > akshare > tushare > pytdx > baostock > yfinance > longbridge |
| LLM统一接口 | LiteLLM 支持 Gemini/Anthropic/OpenAI/DeepSeek |
| 定时任务 | 支持每日定时分析 + 实时推送 |
| Web服务 | FastAPI + React 前端管理界面 |
| 桌面应用 | Electron 打包支持 macOS/Windows |

---

## 2. 项目架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                           用户入口层                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ CLI命令行│  │ Web界面 │  │ 钉钉Bot │  │ 飞书Bot │  │ Discord │  │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  │
└───────┼────────────┼────────────┼────────────┼────────────┼───────┘
        │            │            │            │            │
        ▼            ▼            ▼            ▼            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          API/调度层 (api/)                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ FastAPI Router: /api/v1/analysis | /api/v1/stocks | /api/v1/portfolio │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ main.py: CLI入口、定时调度、线程池调度                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          核心业务层 (src/)                           │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │   Pipeline   │  │   Analyzer   │  │  SearchSvc   │               │
│  │  (调度中心)   │  │  (AI分析器)   │  │  (情报搜索)   │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         │                 │                  │                       │
│  ┌──────┴─────────────────┴──────────────────┴───────┐               │
│  │                   Agent System                      │               │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │               │
│  │  │Technical│ │  Intel  │ │  Risk   │ │Decision │ │               │
│  │  │ Agent   │ │ Agent   │ │ Agent   │ │ Agent   │ │               │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ │               │
│  └──────────────────────────┬──────────────────────────┘               │
│                              │                                        │
│         ┌────────────────────┼────────────────────┐                    │
│         ▼                    ▼                    ▼                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │  Services    │  │ Repositories │  │   Storage    │                 │
│  │  (业务服务)   │  │  (数据访问)   │  │  (ORM模型)   │                 │
│  └──────────────┘  └──────────────┘  └──────────────┘                 │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         数据提供者层 (data_provider/)                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │
│  │ efinance│ │ akshare │ │ tushare │ │ pytdx   │ │ yfinance│        │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘        │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
/workspace/
├── main.py                 # 主入口（CLI + 定时调度）
├── api/
│   ├── app.py             # FastAPI 应用工厂
│   └── v1/
│       ├── endpoints/     # API 路由
│       │   ├── analysis.py
│       │   ├── stocks.py
│       │   ├── portfolio.py
│       │   └── ...
│       └── schemas/       # Pydantic 模型
├── src/
│   ├── agent/             # 多Agent系统
│   │   ├── orchestrator.py    # 流水线编排
│   │   ├── executor.py        # 单Agent执行器
│   │   ├── runner.py          # ReAct执行循环
│   │   ├── llm_adapter.py     # LLM统一接口
│   │   ├── agents/            # 特化Agent
│   │   │   ├── technical_agent.py
│   │   │   ├── intel_agent.py
│   │   │   ├── risk_agent.py
│   │   │   └── decision_agent.py
│   │   ├── tools/             # Agent工具集
│   │   │   ├── registry.py
│   │   │   ├── data_tools.py
│   │   │   ├── analysis_tools.py
│   │   │   └── search_tools.py
│   │   └── skills/            # 技能系统
│   ├── core/              # 核心流水线
│   │   ├── pipeline.py         # StockAnalysisPipeline
│   │   ├── market_review.py    # 大盘复盘
│   │   └── trading_calendar.py # 交易日历
│   ├── services/          # 业务服务
│   │   ├── stock_service.py
│   │   ├── portfolio_service.py
│   │   ├── search_service.py
│   │   └── backtest_service.py
│   ├── repositories/      # 数据访问层
│   ├── notification/      # 通知服务
│   ├── notification_sender/ # 渠道实现
│   ├── config.py          # 配置管理
│   ├── storage.py         # 数据库ORM
│   └── analyzer.py        # AI分析器
├── data_provider/         # 数据源抽象
│   ├── base.py            # DataFetcherManager
│   ├── efinance_fetcher.py
│   ├── akshare_fetcher.py
│   └── ...
├── bot/                   # 机器人平台
│   ├── commands/          # 命令处理
│   └── platforms/         # 平台适配
└── apps/
    ├── dsa-web/          # React前端
    └── dsa-desktop/       # Electron桌面
```

---

## 3. 主要模块职责

### 3.1 API层 (api/)

| 模块 | 文件 | 职责 |
|------|------|------|
| 应用入口 | [api/app.py](file:///workspace/api/app.py) | FastAPI工厂、CORS配置、静态文件服务 |
| 分析接口 | [api/v1/endpoints/analysis.py](file:///workspace/api/v1/endpoints/analysis.py) | `/analyze` 异步分析任务 |
| 股票接口 | [api/v1/endpoints/stocks.py](file:///workspace/api/v1/endpoints/stocks.py) | 股票搜索、实时行情 |
| 组合接口 | [api/v1/endpoints/portfolio.py](file:///workspace/api/v1/endpoints/portfolio.py) | 持仓管理、资金流水 |

### 3.2 核心业务层 (src/)

| 模块 | 文件 | 职责 |
|------|------|------|
| 配置管理 | [src/config.py](file:///workspace/src/config.py) | 单例Config类，100+配置项 |
| 分析流水线 | [src/core/pipeline.py](file:///workspace/src/core/pipeline.py) | 协调数据获取、分析、通知 |
| AI分析器 | [src/analyzer.py](file:///workspace/src/analyzer.py) | GeminiAnalyzer，LLM调用 |
| 通知服务 | [src/notification.py](file:///workspace/src/notification.py) | 多渠道通知分发 |
| 数据库 | [src/storage.py](file:///workspace/src/storage.py) | SQLAlchemy ORM模型 |
| 搜索服务 | [src/search_service.py](file:///workspace/src/search_service.py) | 多搜索引擎统一接口 |

### 3.3 Agent系统 (src/agent/)

| 模块 | 文件 | 职责 |
|------|------|------|
| 流水线编排 | [src/agent/orchestrator.py](file:///workspace/src/agent/orchestrator.py) | 多Agent顺序执行，超时控制 |
| 执行器 | [src/agent/executor.py](file:///workspace/src/agent/executor.py) | 单Agent ReAct循环 |
| 运行器 | [src/agent/runner.py](file:///workspace/src/agent/runner.py) | LLM+工具调用核心循环 |
| LLM适配器 | [src/agent/llm_adapter.py](file:///workspace/src/agent/llm_adapter.py) | LiteLLM统一接口，多Key路由 |
| 工具注册 | [src/agent/tools/registry.py](file:///workspace/src/agent/tools/registry.py) | 工具定义与执行 |

**Agent模式说明**：

```python
# AgentOrchestrator 支持四种执行模式
VALID_MODES = ("quick", "standard", "full", "specialist")

# quick:    Technical → Decision      (~2 LLM调用)
# standard: Technical → Intel → Decision (默认)
# full:     Technical → Intel → Risk → Decision
# specialist: Technical → Intel → Risk → [Skills] → Decision
```

### 3.4 数据提供者层 (data_provider/)

| 文件 | 优先级 | 描述 |
|------|--------|------|
| efinance_fetcher.py | 0 (最高) | 东方财富数据源 |
| akshare_fetcher.py | 1 | 东方财富爬虫 |
| tushare_fetcher.py | 2 | 挖地兔Pro API |
| pytdx_fetcher.py | 2 | 通达信行情 |
| baostock_fetcher.py | 3 | 证券宝 |
| yfinance_fetcher.py | 4 | Yahoo Finance |
| longbridge_fetcher.py | 5 (最低) | 长桥OpenAPI |

### 3.5 机器人层 (bot/)

| 模块 | 描述 |
|------|------|
| platforms/dingtalk_stream.py | 钉钉Stream模式 |
| platforms/feishu_stream.py | 飞书Stream模式 |
| platforms/discord.py | Discord适配 |
| commands/ | 分析、行情、组合等命令 |

---

## 4. 关键类与函数说明

### 4.1 配置模块 (src/config.py)

```python
class Config(metaclass=_Singleton):
    """单例配置类，100+配置项"""
    
    @classmethod
    def get_instance(cls) -> "Config":
        """获取单例实例"""
        
    def validate(self) -> List[str]:
        """验证配置，返回警告列表"""
        
    def refresh_stock_list(self) -> None:
        """从环境变量重新加载股票列表"""
```

**关键配置字段**：

| 字段 | 类型 | 描述 |
|------|------|------|
| `stock_list` | List[str] | 分析股票代码列表 |
| `gemini_api_key` | str | Gemini API密钥 |
| `openai_api_key` | str | OpenAI API密钥 |
| `litellm_model` | str | LLM模型名称 |
| `llm_model_list` | List[dict] | 多通道YAML配置 |
| `max_workers` | int | 并发线程数 |
| `schedule_enabled` | bool | 定时任务开关 |
| `schedule_time` | str | 定时执行时间 |
| `notification_channels` | List[str] | 通知渠道列表 |
| `wechat_webhook_url` | str | 企业微信Webhook |
| `feishu_webhook_url` | str | 飞书Webhook |
| `telegram_bot_token` | str | Telegram Bot Token |
| `realtime_source_priority` | List[str] | 实时行情优先级 |

### 4.2 分析流水线 (src/core/pipeline.py)

```python
class StockAnalysisPipeline:
    """股票分析主流程调度器"""
    
    def __init__(
        self,
        config: Optional[Config] = None,
        max_workers: Optional[int] = None,
        query_id: Optional[str] = None,
    ):
        """初始化调度器"""
        self.config = config
        self.max_workers = max_workers
        self.fetcher_manager = DataFetcherManager()
        self.trend_analyzer = StockTrendAnalyzer()
        self.analyzer = GeminiAnalyzer(config=self.config)
        self.notifier = NotificationService()
        self.search_service = SearchService(...)
    
    def run(
        self,
        stock_codes: Optional[List[str]] = None,
        dry_run: bool = False,
        send_notification: bool = True,
    ) -> List[AnalysisResult]:
        """执行完整分析流程"""
        
    def fetch_and_save_stock_data(
        self,
        code: str,
        force_refresh: bool = False,
    ) -> Tuple[bool, Optional[str]]:
        """获取并保存单只股票数据"""
```

### 4.3 Agent编排器 (src/agent/orchestrator.py)

```python
@dataclass
class OrchestratorResult:
    """多Agent流水线执行结果"""
    success: bool
    content: str
    dashboard: Optional[Dict[str, Any]]
    tool_calls_log: List[Dict[str, Any]]
    total_steps: int
    total_tokens: int
    provider: str
    model: str
    error: Optional[str]
    stats: Optional[AgentRunStats]


class AgentOrchestrator:
    """多Agent流水线编排器"""
    
    def __init__(
        self,
        tool_registry: ToolRegistry,
        llm_adapter: LLMToolAdapter,
        mode: str = "standard",
    ):
        """初始化编排器"""
        
    def run(
        self,
        task: str,
        context: Optional[Dict[str, Any]] = None,
    ) -> AgentResult:
        """执行多Agent流水线（仪表盘模式）"""
        
    def chat(
        self,
        message: str,
        session_id: str,
    ) -> AgentResult:
        """执行多Agent流水线（对话模式）"""
        
    def _execute_pipeline(
        self,
        ctx: AgentContext,
    ) -> OrchestratorResult:
        """核心流水线执行逻辑"""
```

### 4.4 LLM适配器 (src/agent/llm_adapter.py)

```python
@dataclass
class ToolCall:
    """LLM请求的工具调用"""
    id: str
    name: str
    arguments: Dict[str, Any]
    thought_signature: Optional[str] = None


@dataclass
class LLMResponse:
    """LLM响应"""
    content: Optional[str]
    tool_calls: List[ToolCall]
    reasoning_content: Optional[str]  # CoT内容
    usage: Dict[str, Any]             # token统计
    provider: str
    model: str


class LLMToolAdapter:
    """统一LLM接口，支持多Provider"""
    
    def __init__(self, config=None):
        self._router = Router(...)  # LiteLLM Router
        self._litellm_available = True
        
    def call_with_tools(
        self,
        messages: List[Dict[str, Any]],
        tools: List[dict],
        timeout: Optional[float] = None,
    ) -> LLMResponse:
        """发送带工具声明的LLM调用"""
        
    def call_text(
        self,
        messages: List[Dict[str, Any]],
    ) -> LLMResponse:
        """纯文本补全调用"""
```

### 4.5 工具注册器 (src/agent/tools/registry.py)

```python
@dataclass
class ToolParameter:
    """工具参数定义"""
    name: str
    type: str  # "string" | "number" | "integer" | "boolean"
    description: str
    required: bool = True
    enum: Optional[List[str]] = None


@dataclass
class ToolDefinition:
    """工具完整定义"""
    name: str
    description: str
    parameters: List[ToolParameter]
    handler: Callable
    category: str = "data"


class ToolRegistry:
    """Agent工具注册中心"""
    
    def register(self, tool_def: ToolDefinition) -> None:
        """注册工具"""
        
    def execute(self, name: str, **kwargs) -> Any:
        """执行工具"""
        
    def to_openai_tools(self) -> List[dict]:
        """导出OpenAI格式工具声明"""
```

### 4.6 通知服务 (src/notification.py)

```python
class NotificationService:
    """多渠道通知服务"""
    
    # 通知渠道枚举
    class Channel(Enum):
        WECHAT = "wechat"
        FEISHU = "feishu"
        TELEGRAM = "telegram"
        EMAIL = "email"
        DISCORD = "discord"
        SLACK = "slack"
        PUSHOVER = "pushover"
        PUSHPLUS = "pushplus"
        SERVERCHAN3 = "serverchan3"
        CUSTOM = "custom"
        ASTRBOT = "astrbot"
    
    def send(
        self,
        content: str,
        channel: Optional[Channel] = None,
        **kwargs
    ) -> bool:
        """发送通知"""
        
    def generate_daily_report(self, results: List[AnalysisResult]) -> str:
        """生成每日分析报告"""
        
    def generate_dashboard_report(
        self,
        results: List[AnalysisResult],
        report_type: str = "simple",
    ) -> str:
        """生成仪表盘报告"""
        
    def save_report_to_file(
        self,
        content: str,
        filename: str = "analysis_report.md",
    ) -> Optional[str]:
        """保存报告到文件"""
```

### 4.7 数据库存储 (src/storage.py)

```python
# ORM模型定义
class StockDaily(Base):
    """股票每日行情"""
    __tablename__ = "stock_daily"
    
    id = Column(Integer, primary_key=True)
    stock_code = Column(String, nullable=False)
    stock_name = Column(String)
    date = Column(Date, nullable=False)
    open = Column(Float)
    high = Column(Float)
    low = Column(Float)
    close = Column(Float)
    volume = Column(Float)
    amount = Column(Float)
    change_pct = Column(Float)
    created_at = Column(DateTime, default=datetime.utcnow)


class NewsIntel(Base):
    """新闻情报"""
    __tablename__ = "news_intel"
    
    id = Column(Integer, primary_key=True)
    stock_code = Column(String, nullable=False)
    title = Column(String)
    content = Column(Text)
    source = Column(String)
    url = Column(String)
    published_date = Column(DateTime)
    sentiment = Column(String)
    tags = Column(JSON)
    created_at = Column(DateTime, default=datetime.utcnow)


class AnalysisHistory(Base):
    """分析历史"""
    __tablename__ = "analysis_history"
    
    id = Column(Integer, primary_key=True)
    stock_code = Column(String, nullable=False)
    stock_name = Column(String)
    sentiment_score = Column(Integer)
    decision_type = Column(String)  # buy/hold/sell
    operation_advice = Column(String)
    trend_prediction = Column(String)
    report_content = Column(Text)
    dashboard_json = Column(JSON)
    created_at = Column(DateTime, default=datetime.utcnow)


class BacktestResult(Base):
    """回测结果"""
    __tablename__ = "backtest_result"
    
    id = Column(Integer, primary_key=True)
    stock_code = Column(String, nullable=False)
    signal_date = Column(Date, nullable=False)
    decision_type = Column(String)
    sentiment_score = Column(Integer)
    future_close = Column(Float)
    future_change_pct = Column(Float)
    hit = Column(Boolean)
    created_at = Column(DateTime, default=datetime.utcnow)
```

### 4.8 组合服务 (src/services/portfolio_service.py)

```python
class PortfolioService:
    """组合账户业务逻辑"""
    
    def create_account(
        self,
        name: str,
        market: str,
        base_currency: str,
        broker: Optional[str] = None,
    ) -> Dict[str, Any]:
        """创建账户"""
        
    def record_trade(
        self,
        account_id: int,
        symbol: str,
        trade_date: date,
        side: str,  # buy/sell
        quantity: float,
        price: float,
        fee: float = 0.0,
        tax: float = 0.0,
    ) -> Dict[str, Any]:
        """记录交易"""
        
    def get_portfolio_snapshot(
        self,
        account_id: Optional[int] = None,
        as_of: Optional[date] = None,
        cost_method: str = "fifo",
    ) -> Dict[str, Any]:
        """获取组合快照"""
        
    def refresh_fx_rates(
        self,
        account_id: Optional[int] = None,
    ) -> Dict[str, Any]:
        """刷新汇率"""
```

### 4.9 数据提供者 (data_provider/base.py)

```python
class DataFetcherManager:
    """多数据源管理器"""
    
    def get_realtime_quote(self, stock_code: str) -> Optional[UnifiedRealtimeQuote]:
        """获取实时行情"""
        
    def get_daily_data(
        self,
        stock_code: str,
        days: int = 30,
    ) -> Tuple[Optional[pd.DataFrame], Optional[str]]:
        """获取日K线数据"""
        
    def get_chip_distribution(self, stock_code: str) -> Optional[ChipDistribution]:
        """获取筹码分布"""
        
    def get_stock_name(self, stock_code: str) -> Optional[str]:
        """获取股票名称"""
```

---

## 5. 依赖关系

### 5.1 核心依赖

| 包 | 版本 | 用途 |
|----|------|------|
| `python-dotenv` | >=1.0.0 | 环境变量配置 |
| `sqlalchemy` | >=2.0.0 | ORM数据库 |
| `schedule` | >=1.2.0 | 定时任务 |
| `exchange-calendars` | >=4.5.0 | 交易日历 |
| `pandas` | >=2.0.0 | 数据分析 |
| `numpy` | >=1.24.0 | 数值计算 |
| `json-repair` | >=0.55.1 | JSON修复 |

### 5.2 数据源依赖

| 包 | 用途 |
|----|------|
| `efinance>=0.5.5` | 东方财富数据（最高优先级） |
| `akshare>=1.12.0` | 东方财富爬虫 |
| `tushare>=1.4.0` | 挖地兔Pro API |
| `pytdx>=1.72` | 通达信行情 |
| `baostock>=0.8.0` | 证券宝 |
| `yfinance>=0.2.0` | Yahoo Finance |
| `longbridge>=0.2.0` | 长桥OpenAPI |

### 5.3 AI/搜索依赖

| 包 | 用途 |
|----|------|
| `litellm>=1.80.10` | 统一LLM客户端 |
| `tiktoken>=0.8.0` | Token计数 |
| `tavily-python>=0.3.0` | Tavily搜索 |
| `google-search-results>=2.4.0` | SerpAPI搜索 |
| `newspaper3k>=0.2.8` | 文章提取 |
| `jinja2>=3.1.0` | 报告模板引擎 |

### 5.4 Web/通知依赖

| 包 | 用途 |
|----|------|
| `fastapi>=0.109.0` | Web框架 |
| `uvicorn>=0.27.0` | ASGI服务器 |
| `lark-oapi>=1.0.0` | 飞书API |
| `discord.py>=2.0.0` | Discord机器人 |
| `dingtalk-stream>=0.24.3` | 钉钉Stream SDK |

---

## 6. 项目运行方式

### 6.1 命令行运行

```bash
# 正常分析运行
python main.py

# 调试模式（详细日志）
python main.py --debug

# 仅获取数据，不进行AI分析
python main.py --dry-run

# 指定分析特定股票
python main.py --stocks 600519,000001

# 不发送推送通知
python main.py --no-notify

# 单股推送模式（每分析完一只立即推送）
python main.py --single-notify

# 启用定时任务模式
python main.py --schedule

# 仅运行大盘复盘
python main.py --market-review

# 跳过大盘复盘
python main.py --no-market-review

# 强制执行（跳过交易日检查）
python main.py --force-run

# 运行回测
python main.py --backtest
python main.py --backtest --backtest-code 600519 --backtest-days 10
```

### 6.2 Web服务运行

```bash
# 启动Web服务 + 执行分析
python main.py --serve --port 8000

# 仅启动Web服务，不执行分析
python main.py --serve-only

# 指定监听地址
python main.py --serve --host 0.0.0.0 --port 8080

# Web界面
# http://localhost:8000
# API文档: http://localhost:8000/docs
```

### 6.3 API调用示例

```bash
# 健康检查
curl http://localhost:8000/api/v1/health

# 触发分析
curl -X POST http://localhost:8000/api/v1/analysis/analyze \
  -H "Content-Type: application/json" \
  -d '{"stock_codes": ["600519", "000001"]}'

# 查询分析历史
curl http://localhost:8000/api/v1/history?stock_code=600519

# 获取股票实时行情
curl http://localhost:8000/api/v1/stocks/realtime/600519

# 查询组合快照
curl http://localhost:8000/api/v1/portfolio/snapshot
```

### 6.4 Docker部署

```bash
# 构建镜像
docker build -t stock-analysis .

# 运行容器
docker run -d \
  --name stock-analysis \
  -p 8000:8000 \
  -v $(pwd)/data:/app/data \
  -v $(pwd)/.env:/app/.env \
  stock-analysis

# Docker Compose
docker-compose up -d
```

---

## 7. 配置说明

### 7.1 环境变量配置 (.env)

```bash
# ========== 股票列表 ==========
STOCK_LIST=600519,000001,000858

# ========== LLM配置 ==========
# 方式1: 单一模型
GEMINI_API_KEY=your_gemini_api_key
OPENAI_API_KEY=your_openai_api_key
LITELLM_MODEL=gemini/gemini-2.0-flash

# 方式2: 多通道YAML配置
LLM_CONFIG_PATH=/path/to/litellm_config.yaml

# ========== 通知渠道 ==========
NOTIFICATION_CHANNELS=wechat,feishu,telegram
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/...
FEISHU_WEBHOOK_URL=https://open.feishu.cn/...
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# ========== 搜索配置 ==========
TAVILY_API_KEY=your_tavily_api_key
BOCHA_API_KEY=your_bocha_api_key

# ========== 定时任务 ==========
SCHEDULE_ENABLED=true
SCHEDULE_TIME=18:00

# ========== Web服务 ==========
WEBUI_ENABLED=true
WEBUI_HOST=0.0.0.0
WEBUI_PORT=8000
```

### 7.2 LLM通道YAML配置示例

```yaml
# litellm_config.yaml
model_list:
  - model_name: gemini-flash
    litellm_params:
      model: gemini/gemini-2.0-flash
      api_key: os.environ/GEMINI_API_KEY
  
  - model_name: deepseek-chat
    litellm_params:
      model: deepseek/deepseek-chat-v3
      api_key: os.environ/DEEPSEEK_API_KEY
  
  - model_name: anthropic-sonnet
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: os.environ/ANTHROPIC_API_KEY
```

### 7.3 关键配置项速查

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `MAX_WORKERS` | 3 | 并发分析线程数 |
| `ENABLE_REALTIME_QUOTE` | true | 启用实时行情 |
| `ENABLE_CHIP_DISTRIBUTION` | true | 启用筹码分布 |
| `MARKET_REVIEW_ENABLED` | true | 启用大盘复盘 |
| `MERGE_EMAIL_NOTIFICATION` | false | 合并个股+大盘推送 |
| `SINGLE_STOCK_NOTIFY` | false | 单股单独推送 |
| `TRADING_DAY_CHECK_ENABLED` | true | 交易日检查 |
| `BACKTEST_ENABLED` | false | 自动回测 |
| `SAVE_CONTEXT_SNAPSHOT` | true | 保存分析上下文 |

---

## 8. 数据库模型

### 8.1 ER图

```
┌─────────────────┐     ┌─────────────────┐
│   StockDaily    │     │    NewsIntel    │
├─────────────────┤     ├─────────────────┤
│ stock_code (PK) │     │  id (PK)        │
│ date (PK)       │     │  stock_code     │
│ open            │     │  title          │
│ high            │     │  content        │
│ low             │     │  source         │
│ close           │     │  sentiment       │
│ volume          │     └────────┬────────┘
│ amount          │              │
└─────────────────┘              │
                                  │
┌─────────────────┐     ┌────────┴────────┐
│  AnalysisHistory │     │  BacktestResult │
├─────────────────┤     ├─────────────────┤
│  id (PK)        │     │  id (PK)         │
│ stock_code      │─────┤ stock_code      │
│ sentiment_score │     │ signal_date     │
│ decision_type   │     │ decision_type   │
│ dashboard_json  │     │ future_change   │
│ created_at      │     │ hit             │
└─────────────────┘     └─────────────────┘

┌─────────────────────────────────────────────────┐
│              Portfolio Models                    │
├─────────────────┬─────────────────┬──────────────┤
│PortfolioAccount │ PortfolioTrade  │PortfolioPos  │
├─────────────────┼─────────────────┼──────────────┤
│ id (PK)         │ id (PK)         │ id (PK)     │
│ name            │ account_id (FK) │ account_id   │
│ market          │ symbol          │ symbol       │
│ base_currency   │ trade_date      │ quantity     │
│ total_cash      │ side (buy/sell) │ avg_cost    │
│ is_active       │ quantity        │ market_value│
└─────────────────┴─────────────────┴──────────────┘
```

### 8.2 数据库操作接口

```python
# src/storage.py
class DatabaseManager:
    """数据库单例管理"""
    
    @classmethod
    def get_instance(cls) -> "DatabaseManager":
        """获取单例"""
        
    def save_daily_data(
        self,
        stock_code: str,
        data: List[Dict],
    ) -> int:
        """保存日线数据，返回插入数量"""
        
    def get_analysis_history(
        self,
        stock_code: str,
        limit: int = 10,
    ) -> List[AnalysisHistory]:
        """获取分析历史"""
        
    def save_analysis_history(
        self,
        stock_code: str,
        result: AnalysisResult,
    ) -> AnalysisHistory:
        """保存分析结果"""
        
    def save_news_intel(
        self,
        stock_code: str,
        news_list: List[Dict],
    ) -> int:
        """保存新闻情报"""
```

---

## 附录

### A. 决策仪表盘JSON结构

```json
{
  "stock_name": "贵州茅台",
  "sentiment_score": 75,
  "trend_prediction": "看多",
  "operation_advice": "买入",
  "decision_type": "buy",
  "confidence_level": "高",
  "dashboard": {
    "core_conclusion": {
      "one_sentence": "MA多头排列，量价配合良好，可买入",
      "signal_type": "🟢买入信号",
      "time_sensitivity": "本周内",
      "position_advice": {
        "no_position": "可结合支撑位分批试仓",
        "has_position": "继续持有，回踩关键位加仓"
      }
    },
    "data_perspective": {
      "trend_status": {
        "ma_alignment": "bullish",
        "is_bullish": true,
        "trend_score": 8
      },
      "price_position": {
        "current_price": 1850.0,
        "ma5": 1830.0,
        "support_level": 1800.0
      },
      "volume_analysis": {
        "volume_ratio": 1.5,
        "volume_status": "放量上涨"
      }
    },
    "intelligence": {
      "risk_alerts": ["股东减持公告"],
      "positive_catalysts": ["业绩预增"]
    },
    "battle_plan": {
      "sniper_points": {
        "ideal_buy": 1820.0,
        "stop_loss": 1780.0,
        "take_profit": 1950.0
      }
    }
  },
  "analysis_summary": "综合技术面和消息面分析...",
  "risk_warning": "注意大盘系统性风险"
}
```

### B. 错误码说明

| 错误码 | 含义 | 处理建议 |
|--------|------|----------|
| E001 | LLM调用失败 | 检查API Key配置，启用降级 |
| E002 | 数据获取失败 | 检查网络，切换备用数据源 |
| E003 | 通知发送失败 | 检查Webhook配置 |
| E004 | 交易日检查跳过 | 使用 --force-run 强制执行 |
| E005 | 超时中断 | 增加 timeout 配置 |

### C. 日志文件位置

```
logs/
├── stock_analysis.log      # 主日志
├── stock_analysis.debug.log # 调试日志（--debug模式）
└── error.log               # 错误日志
```

---

*文档版本: 1.0.0*
*最后更新: 2026-05-08*
*项目: A股自选股智能分析系统*
