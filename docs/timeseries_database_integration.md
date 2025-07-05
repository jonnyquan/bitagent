# BitAgent时序数据库集成

## 概述

BitAgent时序数据库集成提供了高性能的历史数据存储和查询功能，支持大规模数据分析和回测优化。系统设计了灵活的抽象层，可以在InfluxDB和内存数据库之间无缝切换。

## 核心特性

### 🏗️ 架构特点
- **统一接口**: 抽象化的时序数据库接口，支持多种后端
- **高性能**: 批量写入和查询优化，支持并发操作
- **智能缓存**: 多层缓存策略，提升查询性能
- **自动迁移**: 完整的数据迁移工具和向后兼容

### 📊 数据类型支持
- **市场数据**: 价格、成交量、技术指标
- **策略信号**: 买卖信号、置信度、元数据
- **组合指标**: 总价值、盈亏、持仓分布
- **风险指标**: VaR、夏普比率、最大回撤

### ⚡ 性能优化
- **批量操作**: 1000-5000 points/sec 写入性能
- **查询优化**: 毫秒级响应时间
- **内存管理**: 智能缓存和过期清理
- **连接池**: 高并发场景下的连接复用

## 快速开始

### 1. 安装依赖

```bash
# 基础功能
pip install pandas numpy asyncio

# InfluxDB支持（可选）
pip install influxdb-client

# 监控支持
pip install prometheus-client
```

### 2. 基础使用

```python
import asyncio
from datetime import datetime, timezone
from src.core.timeseries_db import create_timeseries_manager

async def basic_example():
    # 创建时序数据库管理器
    ts_manager = await create_timeseries_manager(
        use_influxdb=False,  # 使用内存数据库进行测试
    )
    
    try:
        # 写入市场数据
        await ts_manager.write_market_data(
            symbol='BTCUSDT',
            timestamp=datetime.now(timezone.utc),
            price=45000.0,
            volume=1000000.0,
            source='binance'
        )
        
        # 查询市场数据
        df = await ts_manager.get_market_data(
            symbol='BTCUSDT',
            start_time=datetime.now(timezone.utc) - timedelta(hours=24),
            source='binance'
        )
        
        print(f"Retrieved {len(df)} records")
        
    finally:
        await ts_manager.stop()

# 运行示例
asyncio.run(basic_example())
```

### 3. InfluxDB配置

```python
# 生产环境配置
ts_manager = await create_timeseries_manager(
    use_influxdb=True,
    influx_url="http://your-influxdb:8086",
    influx_token="your-token",
    influx_org="your-org",
    influx_bucket="trading_data"
)
```

## 数据模型

### 市场数据 (market_data)
```
measurement: market_data
tags:
  - symbol: 交易对 (BTCUSDT, ETHUSDT)
  - source: 数据源 (binance, coinbase)
fields:
  - price: 价格 (float)
  - volume: 成交量 (float)
timestamp: 时间戳
```

### 策略信号 (strategy_signals)
```
measurement: strategy_signals
tags:
  - strategy: 策略名称 (rsi_strategy, bb_squeeze)
  - symbol: 交易对
  - signal: 信号类型 (BUY, SELL, HOLD)
fields:
  - confidence: 置信度 (0.0-1.0)
  - [metadata]: 额外指标
timestamp: 时间戳
```

### 组合指标 (portfolio_metrics)
```
measurement: portfolio_metrics
tags:
  - strategy: 策略名称
fields:
  - total_value: 总价值
  - pnl: 盈亏
  - position_count: 持仓数量
timestamp: 时间戳
```

### 风险指标 (risk_metrics)
```
measurement: risk_metrics
tags:
  - strategy: 策略名称
fields:
  - var_1d: 1日VaR
  - sharpe_ratio: 夏普比率
  - max_drawdown: 最大回撤
timestamp: 时间戳
```

## API参考

### TimeSeriesDBManager

#### 写入方法

```python
# 写入市场数据
await ts_manager.write_market_data(
    symbol: str,
    timestamp: datetime,
    price: float,
    volume: float,
    source: str = "binance"
)

# 写入策略信号
await ts_manager.write_strategy_signal(
    strategy_name: str,
    symbol: str,
    timestamp: datetime,
    signal: str,
    confidence: float,
    metadata: Optional[Dict[str, Any]] = None
)

# 写入组合指标
await ts_manager.write_portfolio_metrics(
    timestamp: datetime,
    total_value: float,
    pnl: float,
    positions: Dict[str, float],
    strategy: str = "all"
)

# 写入风险指标
await ts_manager.write_risk_metrics(
    timestamp: datetime,
    var_1d: float,
    sharpe_ratio: float,
    max_drawdown: float,
    strategy: str
)
```

#### 查询方法

```python
# 查询市场数据
df = await ts_manager.get_market_data(
    symbol: str,
    start_time: datetime,
    end_time: Optional[datetime] = None,
    source: str = "binance"
) -> pd.DataFrame

# 查询策略信号
df = await ts_manager.get_strategy_signals(
    strategy_name: str,
    start_time: datetime,
    end_time: Optional[datetime] = None,
    symbol: Optional[str] = None
) -> pd.DataFrame

# 查询组合表现
df = await ts_manager.get_portfolio_performance(
    start_time: datetime,
    end_time: Optional[datetime] = None,
    strategy: str = "all"
) -> pd.DataFrame
```

### EnhancedDataAgent

增强数据代理提供消息总线集成和缓存功能：

```python
from src.agents.enhanced_data_agent import EnhancedDataAgent

# 创建代理
agent = EnhancedDataAgent(
    bus=message_bus,
    use_influxdb=True,
    cache_duration=300  # 5分钟缓存
)

# 启动代理
await agent.startup()

# 通过消息总线查询数据
result = await bus.request("data.query.market", {
    'symbol': 'BTCUSDT',
    'start_time': start_time.isoformat(),
    'use_cache': True
})
```

## 数据迁移

### 自动迁移工具

```python
from src.core.data_migration import run_migration

# 运行完整迁移
await run_migration(
    use_influxdb=True,
    influx_url="http://localhost:8086",
    influx_token="your-token",
    include_sample_data=True
)
```

### 手动迁移

```python
from src.core.data_migration import DataMigrationTool

ts_manager = await create_timeseries_manager(use_influxdb=True)
migration_tool = DataMigrationTool(ts_manager)

# 迁移特定目录
await migration_tool.migrate_directory_data(
    source_dir=Path("./data"),
    batch_size=1000
)

# 迁移缓存数据
await migration_tool._migrate_cache_data()
```

## 性能调优

### 批量写入优化

```python
# 配置批量写入参数
ts_manager = TimeSeriesDBManager(db)
ts_manager._batch_size = 5000      # 增加批量大小
ts_manager._batch_timeout = 10.0   # 延长超时时间

# 手动刷新缓冲区
await ts_manager._flush_batch()
```

### 查询优化

```python
# 使用字段过滤
options = QueryOptions(
    measurement="market_data",
    start_time=start_time,
    end_time=end_time,
    fields=["price", "volume"],  # 只查询需要的字段
    limit=1000                    # 限制返回数量
)

# 使用聚合查询
options.aggregate = "mean"
options.group_by = QueryPeriod.HOUR
```

### 缓存策略

```python
# 配置缓存参数
agent = EnhancedDataAgent(
    bus=bus,
    cache_duration=600,          # 10分钟缓存
    max_query_range_days=30      # 限制查询范围
)

# 手动清理缓存
await agent._cleanup_expired_cache()
```

## 监控和告警

### Prometheus指标

系统自动暴露以下Prometheus指标：

```
# 查询指标
enhanced_data_queries_total{query_type="market_data"}
enhanced_data_query_duration_seconds{query_type="market_data"}

# 写入指标
enhanced_data_points_written_total
enhanced_data_cache_hits_total
enhanced_data_cache_misses_total

# 连接指标
enhanced_data_active_connections
```

### 健康检查

```python
# 检查系统健康状态
health_status = await agent._handle_health_check({})
print(health_status['status'])  # healthy, degraded, unhealthy
```

### 性能统计

```python
# 获取性能统计
stats = await agent._handle_stats_request({})
print(f"Cache hit rate: {stats['query_stats']['cache_hit_rate']:.2%}")
```

## 配置管理

### 配置文件

系统支持YAML配置文件 (`config/timeseries_db.yaml`)：

```yaml
database:
  use_influxdb: true
  influxdb:
    url: "http://localhost:8086"
    token: "${INFLUXDB_TOKEN}"
    org: "bitagent"
    bucket: "trading_data"

cache:
  duration_seconds: 300
  max_cache_size: 1000

query:
  max_range_days: 90
  default_limit: 10000
```

### 环境变量

```bash
export INFLUXDB_TOKEN="your-influxdb-token"
export INFLUXDB_URL="http://your-influxdb:8086"
export BITAGENT_DB_CACHE_DURATION=300
export BITAGENT_DB_BATCH_SIZE=1000
```

## 最佳实践

### 1. 数据分区策略

```python
# 按时间范围分区查询
async def query_with_partitioning(ts_manager, symbol, days=30):
    results = []
    end_time = datetime.now(timezone.utc)
    
    for i in range(0, days, 7):  # 每7天一个分区
        start_time = end_time - timedelta(days=min(7, days-i))
        
        df = await ts_manager.get_market_data(
            symbol=symbol,
            start_time=start_time,
            end_time=end_time
        )
        results.append(df)
        end_time = start_time
    
    return pd.concat(results, ignore_index=True)
```

### 2. 错误处理和重试

```python
import asyncio
from typing import Optional

async def robust_write(ts_manager, data, max_retries=3):
    for attempt in range(max_retries):
        try:
            await ts_manager.write_market_data(**data)
            return True
        except Exception as e:
            if attempt == max_retries - 1:
                logger.error(f"Write failed after {max_retries} attempts: {e}")
                return False
            
            await asyncio.sleep(2 ** attempt)  # 指数退避
    
    return False
```

### 3. 内存管理

```python
# 定期清理过期数据
async def cleanup_old_data(ts_manager, days_to_keep=30):
    cutoff_date = datetime.now(timezone.utc) - timedelta(days=days_to_keep)
    
    measurements = ["market_data", "strategy_signals", "portfolio_metrics"]
    
    for measurement in measurements:
        await ts_manager.db.delete_measurement(
            measurement=measurement,
            start_time=datetime(2020, 1, 1, tzinfo=timezone.utc),
            end_time=cutoff_date
        )
```

## 故障排除

### 常见问题

1. **InfluxDB连接失败**
   ```bash
   # 检查InfluxDB是否运行
   curl -I http://localhost:8086/health
   
   # 验证token和权限
   influx auth list
   ```

2. **查询返回空结果**
   ```python
   # 检查时间范围和时区
   start_time = datetime.now(timezone.utc) - timedelta(hours=1)
   
   # 验证measurement和tags
   options.tags = {"symbol": "BTCUSDT", "source": "binance"}
   ```

3. **性能问题**
   ```python
   # 减少查询范围
   options.limit = 1000
   
   # 启用字段过滤
   options.fields = ["price"]
   
   # 使用缓存
   query_payload['use_cache'] = True
   ```

### 调试工具

```python
# 启用详细日志
import logging
logging.getLogger('src.core.timeseries_db').setLevel(logging.DEBUG)

# 查看查询执行计划
options.debug = True
df = await ts_manager.db.query(options)

# 监控系统资源
import psutil
print(f"Memory usage: {psutil.Process().memory_info().rss / 1024 / 1024:.1f} MB")
```

## 扩展开发

### 自定义数据类型

```python
from src.core.timeseries_db import DataPoint

# 创建自定义数据点
custom_point = DataPoint(
    measurement="custom_metrics",
    timestamp=datetime.now(timezone.utc),
    fields={
        "custom_value": 42.0,
        "custom_flag": True
    },
    tags={
        "environment": "production",
        "version": "1.0"
    }
)

await ts_manager.db.write_point(custom_point)
```

### 自定义查询

```python
# 复杂聚合查询
options = QueryOptions(
    measurement="market_data",
    start_time=start_time,
    end_time=end_time,
    aggregate="mean",
    group_by=QueryPeriod.HOUR,
    tags={"symbol": "BTCUSDT"}
)

df = await ts_manager.db.query(options)
```

### 插件扩展

```python
class CustomTimeSeriesDB(TimeSeriesDB):
    """自定义时序数据库实现。"""
    
    async def connect(self):
        # 自定义连接逻辑
        pass
    
    async def write_point(self, point: DataPoint) -> bool:
        # 自定义写入逻辑
        pass
    
    async def query(self, options: QueryOptions) -> pd.DataFrame:
        # 自定义查询逻辑
        pass
```

## 更新日志

### v1.0.0 (PHASE-3-001)
- ✅ 初始时序数据库集成
- ✅ InfluxDB和内存数据库支持
- ✅ 增强数据代理实现
- ✅ 数据迁移工具
- ✅ 性能优化和缓存
- ✅ 监控和告警集成
- ✅ 完整的测试套件

### 后续计划
- 🔄 多数据源聚合查询
- 🔄 实时数据流处理
- 🔄 分布式存储支持
- 🔄 机器学习特征工程集成