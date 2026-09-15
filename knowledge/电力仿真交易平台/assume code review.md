# 一次市场交易
市场开放
   ↓
UnitsOperator 收到市场开放消息
   ↓
机组/策略生成报价
   ↓
UnitsOperator 提交 orderbook
   ↓
Market 清算
   ↓
返回 accepted_orders / rejected_orders
   ↓
机组更新实际出力
   ↓
计算现金流和奖励
   ↓
进入下一个时间步

World
├── 仿真时钟 Clock
├── 消息运行环境 Container
├── 市场运营商 MarketOperator
│   └── 市场 MarketRole
├── 机组运营商 UnitsOperator
│   └── 发电、负荷、储能等 Unit
├── 输出模块 WriteOutput
└── 强化学习模块 Learning（可选）

|属性|内容|
|---|---|
|`market_operators`|市场运营商 Agent|
|`markets`|市场配置|
|`unit_operators`|发电商、储能商等运营商|
|`units`|电厂、负荷、储能等具体单元|

