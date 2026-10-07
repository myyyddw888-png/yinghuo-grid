# 萤火网格1.0 策略规则详解

## 定位

针对**手工锚仓**的永续合约网格系统。核心思想：用户手动开一个「锚仓」定方向，系统在锚仓基础上挂一层 ATR 动态网格，用马丁递增摊薄成本、用单格止盈保存利润、用两档总止盈止损控制整体风险，全自动闭环执行。

## 参数总表

| 参数 | 值 | 说明 |
|---|---|---|
| `ATR_TIMEFRAME` | 4h | ATR 时间框架 |
| `ATR_PERIOD` | 14 | ATR 周期 |
| `ATR_MULT` | 1.0 | 默认网格间距 = 1×ATR（BTC 单独 0.5） |
| `MIN_SPACING_PCT` | 0.003 | 间距下限 = 0.3%×价格（防低波动频繁触发） |
| `FOLLOW_MULT` | 1.0 | 防踏空：网格清空且偏离 1×ATR 时上移基准 |
| `FIRST_NOTIONAL` | 20.0 | 马丁首笔基准名义（U） |
| `MARTINGALE_TIER1` | 1.5 | 前 3 格马丁系数 |
| `MARTINGALE_TIER2` | 1.7 | 第 4 格起马丁系数 |
| `TIER1_LEVELS` | 3 | 前 3 格用 1.5 |
| `MAX_LEVELS` | 5 | 最多加 5 格（第 6 格封顶） |
| `MIN_NOTIONAL` | 5.0 | 最小下单名义 |
| `GLOBAL_ADD_STOP_PNL` | -0.20 | 账户总浮亏 ≥20% 暂停所有新加仓 |
| `TREND_EMA_PERIOD` | 113 | 趋势线 EMA（4h 框架 ≈18.9 天） |

## 止盈止损档位

- 总止盈：浮盈 ≥20% 减 50% 仓 → ≥30% 减 100% 全平
- 总止损：浮亏 ≥15% 减 50% 仓 → ≥25% 减 100% 全平
- 盈亏按「锚仓 + 网格仓」的加权平均成本计算
- 减仓比例 = 当前实际持仓的百分比（止盈 20% 减 50% 剩 50%）

## 优先级顺序

1. 总止盈止损（两档，最高优先）
2. 单格止盈（LIFO 减最近一格）
3. 马丁加仓（受账户熔断 + 趋势过滤器双重门控）

## 马丁基数三模式

- `fixed`：固定 20U 基准（与锚仓实际大小解耦）
- `anchor`：以锚仓实际名义为基数
- `custom`：自定义数值（如 BTC 用 90U）

## 趋势过滤器

- 趋势线 = 4h EMA113（≈18.9 天，对齐回测的 1h EMA453）
- 做多 + 跌破趋势线 → 暂停加仓；做空 + 站上趋势线 → 暂停加仓
- 回测结论：高波动币（如 XRP）挂过滤器收益巨大（回撤 25%→9%）；低波动币（BTC/ETH）不划算
- 默认策略：**除 BTC/ETH 外全挂**
- 带缓存（4h bar 内不重拉），零额外 API 负担

## 仓位流水记录器

每次操作自动追加 `.manual_grid_journal.jsonl`（jsonl 追加式，永久累积）：

- `add` / `reduce_one`（单格止盈）/ `reduce_pct`（总止盈止损）/ `takeover`（接管）/ `clear`（清仓）
- 每条含 `ts/time/action/symbol/side/price/qty/level_id/notional/entry/realized_pnl`
- 减仓盈亏按 LIFO 精确计算（含锚仓部分）
- 用途：可复盘、审计、三方对账（状态文件 vs 交易所 vs 流水）

## 文件说明

- `scripts/manual_grid_atr.py` — 核心策略脚本（自包含，仅依赖 ccxt）
- `scripts/grid_trend_filter_backtest.py` — 趋势过滤器回测脚本
- 状态文件 `.manual_grid_atr.json`（运行时生成，含 grids/ignored）
- 流水文件 `.manual_grid_journal.jsonl`（运行时生成）
- 告警日志 `.manual_grid_alerts.log`（运行时生成，心跳读取推送）
