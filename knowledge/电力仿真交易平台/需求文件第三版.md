在第二版电力现货市场仿真平台上增加 `household_energy_optimization`。居民根据**开市前可用的预测电价**，在满足设备和用能约束的前提下计算每个交付时段的购电量，再向 EOM 提交买单。预测电价用于制定计划；实际成交价格和成交量由市场出清决定。

在本平台中放置独立的 `examples/input/example_01h`。本版必须跑通其中的 `A360`：一个建筑由 `heat_pump` 和 `generic_storage` 两行组成，生成**一个建筑购电参与者**。电池是建筑内部设备，不再作为第二版独立储能参与者重复报价。

本版的设备验收范围是**热泵和电池**。固定用电负荷仅支持默认值 `P_fixed=0 MW`，尚未验证非零固定负荷。热储能和电动车需要独立输入、设备约束及测试场景；`example_01h` 的 `A360` 不包含这两类设备，不以该例宣称已验证它们。

每个产品对应 **15分钟交付时段**，而非之前版本的1小时。**(config.yaml)**
## 时间、市场和输入

|             输入              |                                                                             第三版要求                                                                             |
| :-------------------------: | :-----------------------------------------------------------------------------------------------------------------------------------------------------------: |
|        `config.yaml`        |                               读取 `eom`：15 分钟步长，按 24 小时开市，一次最多 96 个 15 分钟产品，首个产品从开市后 15 分钟开始，市场机制为 `pay_as_bid`。仅生成处于仿真时间范围内的产品。                               |
|     `demand_units.csv`      |                                                  `demand_EOM` 是独立的普通负荷；注意：读取 `price=62.88 EUR/MWh` 作为最高愿付价。                                                   |
|       `demand_df.csv`       |                                                             `demand_EOM` 列提供各时段普通负荷，本例为 `100 MW`。                                                             |
|     `forecasts_df.csv`      |           `price_EOM` 为 EOM 预测电价，如 `130.92 EUR/MWh`；`A360_heat_demand` 为建筑热需求，如 `3 MW_th`。预测价不作为本次实际出清价。预测电价用于影响 A360 的内部优化决策，主要决定电池什么时候充电、什么时候放电：            |
| `residential_dsm_units.csv` | 按 `name=A360` 合并两行，建筑级字段读取 `bidding_EOM`、`objective`、`flexibility_measure`、`cost_tolerance`、`node`、`unit_operator`，设备级字段分别归属热泵和电池。空单元格表示该设备未配置该参数，不能统一当作数值 0。 |
|        发电机组、燃料和可用率文件        |                                                                沿用第二版供给报价功能，与建筑及普通负荷的买单进入同一场市场。                                                                |
加载时按配置的时区和时间步长对齐全部序列；缺少必需时段、重复时间戳、非法数值、缺少预测价或热需求时，指出文件、字段和时段。第二版按小时聚合的读取逻辑需改为按场景时间步长处理：本例原始 15 分钟数据保持 15 分钟，旧小时场景仍按原规则聚合。对于示例中未被使用的建筑列，给出简明提示，不据此创建 `A361`、`A362`。

统一单位：设备功率用 MW，电池容量用 MWh，价格用 EUR/MWh。时长为 $Δt=0.25 h$ 的产品，其申报电量为 $功率 × 0.25$ MWh；不得把 15 分钟的 MW 数值直接当成 MWh。

统一订单方向与符号：所有买单和卖单的 `volume_mwh` 均保存非负电量，使用 `side=buy` 或 `side=sell` 表示订单方向。进入市场供需平衡计算时，卖单电量取正值，买单电量取负值；该符号转换只用于内部平衡计算，不改变订单中保存的 `volume_mwh`。`demand_results.csv` 中的申报、成交和未成交电量以及 `trade_results.csv` 中的成交电量均使用非负值。

时间序列的时间戳表示交付时段起点。市场在00:00开市时，首个产品为00:15—00:30，读取00:15对应的数据；00:00数据只用于初始状态或历史状态初始化，不生成交付订单。

## 输出结果
保留第二版已有输出，本版新增以下内容。
### demand_results.csv
增加 A360 的买单、成交量和实际支付。市场成交量不得与其他结果重复累加。
### trade_results.csv
记录 `pay_as_bid` 的逐笔成交：
```
delivery_start
delivery_end
opening_id
buyer_name
buyer_bid_id
seller_name
seller_offer_id
trade_energy_mwh
trade_price_eur_per_mwh
payment_eur
```
成交电量使用正值：$$
\mathrm{payment} = \mathrm{trade\_energy} \times \mathrm{trade\_price}$$
`demand_results.csv` 中每张买单的 `payment_eur`，必须等于 `trade_results.csv` 中相同 `opening_id` 和 `buyer_bid_id` 的全部逐笔支付之和。
### household_results.csv
记录 A360 的计划及实际设备运行：
```
delivery_start
delivery_end
unit_name
forecast_price_eur_per_mwh
heat_demand_mw_th
fixed_power_mw
planned_grid_power_mw
heat_pump_power_mw
battery_charge_power_mw
battery_discharge_power_mw
soc_before
soc_after
unmet_electricity_mwh
unmet_heat_mwh_th
```
其中，设备功率和 SOC 均为成交后的实际运行结果。买单、成交量、报价及支付统一记录在 `demand_results.csv`，不在此重复保存。
### household_flexibility_results.csv
记录各时段的购电功率范围：
```
delivery_start
delivery_end
unit_name
minimum_grid_power_mw
maximum_grid_power_mw
```
基准功率读取 `household_results.csv` 中的 `planned_grid_power_mw`；灵活性分析结果不直接用于 EOM 买单。
## demand and supply
### demand
- demand_df: 
	A360_building_load_profile, A360_building_heat_demand, A361_building_load_profile, A361_building_pv_power_profile, A361_building_heat_demand, A362_building_load_profile 不需要考虑。仍要读取 `demand_EOM`
- forecasts_df:
	A360_heat_demand 热泵每消耗 `1 MW` 电功率，可提供 `2 MW` 热功率。所以满足 `3 MW` 热需求时，热泵需用电 `1.5 MW`。(residential_dsm_units.csv)
- residential_dsm_units:
	`generic_storage` 中配置了电池的**额定容量** `0.0362 MWh`（36.2 kWh）、**初始 SOC** `0.0414364`（约 4.14%），以及充电和放电效率 `0.9731`（均为 97.31%）。据此，电池的初始储电量约为 `0.0015 MWh`（1.5 kWh）
$$P_{\text{购电}} = P_{\text{基础用电}} + P_{\text{热泵}} + P_{\text{电池充电}} - P_{\text{电池放电}}.$$
### supply
`household_energy_optimization` 优化的是 **A360 的购电计划**，不优化发电机报价。发电机按各自策略提交卖单，再与 `demand_EOM` 和 A360 的买单一起出清。

电池不动作时，A360 该时段计划购买 `1.5 × 0.25 = 0.375 MWh`；充电会增加计划购电量，放电会减少它。要知道**完整的需求计划**，需要运行优化，并查看它为每个时段生成的购电订单。

`example_01h` 的 EOM 发电机卖单采用可分割方式，具体规则见“发电机可分割卖单”。
## 报价策略 powerplant_energy_naive_otc
该策略用于发电机组参与长期场外交易市场 `LTM_OTC`。策略按照机组边际成本报价，并通过 `scale_firm_power_capacity` 控制投入 OTC 市场的容量比例。
### 输入参数 
- `max_power`：机组额定最大出力。
- `availability`：当前时段可用率；没有对应机组可用率序列时默认值为 `1.0`。 
- `energy`：该交付时段已经承诺的发电功率。 
- `capacity_pos`：已经预留的正备用容量。
- `scale_firm_power_capacity`：OTC 报价容量比例，默认值为 `1.0`。 
- `fuel_price`：燃料价格。 
- `co2_price`：碳排放价格。 
- `efficiency`：机组额定效率。 
- `emission_factor`：燃料排放系数。
- `additional_cost`：附加边际成本。
### 报价量
首先计算机组的实际可用上限：
$$P_t^{available} = availability_t \times P^{max}.$$
扣除该时段已经承诺的发电功率和正备用容量：$$P_t^{remaining} = \max\left( 0,\, P_t^{available} -P_t^{committed} -P_t^{capacity\_pos} \right).$$OTC 报价功率为：$$Q_t^{OTC} = P_t^{remaining} \times scale\_firm\_power\_capacity.$$其中，$Q_t^{OTC}$ 的单位是 MW。
- `1.0`：全部剩余容量参与 OTC 报价；
- `0.5`：50% 的剩余容量参与报价；
- `0`：不提交 OTC 报价。
### 报价价格
假设报价全部成交，机组的最终出力为：
$$P_t^{final} = P_t^{committed}+Q_t^{OTC}.$$
报价价格取最终出力下的边际成本：$$p_t^{OTC} = MC_t(P_t^{final}). $$边际成本计算为：$$MC_t(P_t^{final}) = \frac{\mathrm{fuel\_price}_t}{\eta_t} + \frac{\mathrm{co2\_price}_t \times \mathrm{emission\_factor}}{\eta_t} + \mathrm{additional\_cost}.$$其中，实际效率为$$r_t=\frac{P_t^{final}}{P^{max}}.$$ $\eta_t$ 根据机组最终出力率计算$\eta_t=\eta(r_t)$。（可以参考一下 assume/units/powerplant.py 中calc_marginal_cost_with_partial_eff 函数的逻辑）
### 订单生成
每个 OTC 产品生成一笔卖单，包含：
- `start_time` 
- `end_time` 
- `side=sell`
- `price` 
- `volume_mwh=Q_t^{OTC}\times\Delta t`
- `node` 
其中，$\Delta t$ 是产品时长。15 分钟产品取 $\Delta t=0.25h$，因此 `volume_mwh=Q_t^{OTC}×0.25`。
当报价量为 `0` 时，不提交订单。
### 策略限制
该策略： 
- 不在报价中分摊启动成本；
- 不进行多时段运行优化；
- 不调用逐时段爬坡约束计算； 
- 只在配置了 `LTM_OTC` 市场时执行。 
当前 `example_01h` 只配置了 `EOM`，因此若第三版不新增 `LTM_OTC` 市场，该策略仅作为输入字段保留，不参与实际仿真。
## 优化模型 household_energy_optimization
### 目标与时序
在给定预测覆盖的规划时段中，最小化预计总购电成本：
$$
\min\sum_t \hat\lambda_t P_{\text{购电}}\Delta t,  
\quad  
P_{\text{购电}} = P_{\text{基础用电}} + P_{\text{热泵}} + P_{\text{电池充电}} - P_{\text{电池放电}}. 
$$
$is_{prosumer}=No$ 时不允许净卖电。普通负荷 `demand_EOM` 与 `A360` 分开建模和计量。

1. 每次开市时，根据最新真实 SOC 和当时可获得的预测，优化本次最多 96 个新产品的购电计划，不得读取未来实际出清价。若存在此前已出清但尚未交付的产品，应将其作为不可修改的已知约束，并推算新产品开始交付时的 SOC。
2. 市场关闭后，对本次全部产品一次性出清。成交量、成交价格及拒绝结果随即锁定，不得重新报价、撤销或修改。
3. 出清后，可以根据实际成交量重新安排建筑内部的热泵和电池运行，但不得改变市场订单及成交结果。每个产品交付后按实际运行更新真实 SOC；下一次开市只规划新的产品。

### 热泵与供热
满足各时段热需求：$P_{hp,t} × COP ≥ Q_{heat,t}$，如无蓄热设备且只有热泵供热，取等式。A360 的 `COP=2`、热需求 `3 MW_th`，因此该时段热泵需耗电 `1.5 MW`，对应 15 分钟电量 `0.375 MWh`。受 $0 ≤ P_hp ≤ 5 MW$、每时段爬坡上/下限 `5 MW` 约束。

此例**没有热储能**：热需求必须在对应时段得到满足，不能仅因未来电价便宜就把本时段供热移走。`min_operating_time=0` 不施加最低连续运行约束；以后该字段非零时，须明确按小时换算为 15 分钟步数。
### 电池
电池容量 `C=0.0362 MWh`；`min_soc=initial_soc=0.0414364` 为 **0～1 的比例**；当前输入未提供 `max_soc` 时默认取 `1.0`；充电和放电效率均为 `0.9731`。定义充电功率 `P_charge≥0`、放电功率 `P_discharge≥0`，每时段不得同时充放电，且：$$SOC_{t+1}=SOC_t+  
\frac{\eta_c P_t^{charge}\Delta t-P_t^{discharge}\Delta t/\eta_d}{C},  
\qquad SOC_{min}\leq SOC_t\leq SOC_{max}.$$本例未填 `storage_loss_rate`，默认 0；若以后配置非零损耗，应按交付时长更新状态。将 CSV 中 `max_charging_rate=0.002`、`max_discharging_rate=0.002` **显式映射为最大充/放电功率（MW）**，同时施加两方向的 `ramp_up=ramp_down=0.002` 相邻时段变化限制。需注意：ASSUME 当前通用储能组件并未直接用这两个 CSV 名称设定功率上限；本平台的映射是有意明确设备的物理边界，不能只传入字段却让它失效。

为避免每次规划在最后一个产品人为放空电池，本次规划窗口末端的 SOC 不得低于本次新产品开始交付时推算的初始 SOC：
$$
SOC_{end}\geq SOC_{planning\_start}.
$$
### 成本容忍度与灵活性
`objective=min_variable_cost` 得到上面的最低预测成本计划。`flexibility_measure=cost_based_load_shift` 与 `cost_tolerance=10` 用于**另外评估**满足物理约束的负荷转移空间：基准成本为非负时，新计划成本不得高于基准的 `110%`。若支持负成本场景，容忍度按基准成本绝对值加到基准成本上，避免负数乘以 `1.1` 反而缩紧上限。

灵活性评估不能移动 `P_fixed`，也不能越过供热、SOC 或设备功率边界；其结果单独报告。EOM 买单使用最低成本计划 `P_grid`，不把灵活性分析得到的另一套用电量直接当作购电订单。
## 发电机可分割卖单
`example_01h` 的 EOM 使用 `pay_as_bid`，发电机对每个 15 分钟产品提交普通可分割卖单。卖单可在 `0` 到全部申报电量之间成交，边际卖单允许部分成交。

机组在时段 $t$ 的可申报功率上限为：
$$
P_t^{offer}=\max\left(0,\ \mathrm{availability}_t P^{\max}-P_t^{committed}-P_t^{capacity\_pos}\right).
$$
其中，$P_t^{committed}$ 是已经在其他市场承诺的发电功率，$P_t^{capacity\_pos}$ 是已经预留的正备用容量；没有相关市场承诺时二者均为 `0`。没有对应机组可用率序列时，$\mathrm{availability}_t$ 默认取 `1.0`。15 分钟产品的申报电量为：
$$
E_t^{offer}=P_t^{offer}\times 0.25\ \mathrm{MWh}.
$$

该场景中的 `min_power` 不作为卖单的最小接受量。`ramp_up`、`ramp_down`、`min_operating_time`、`min_down_time` 和启动成本不进入 `example_01h` 的 EOM 报价、出清及结算。市场可以接受低于 `min_power` 的功率，实际成交功率为：
$$
P_t^{accepted}=\frac{E_t^{accepted}}{0.25}.
$$
这样，NEURATH F 的 `min_power=560 MW` 不会要求负荷同时达到 560 MW。该处理是本场景采用的市场抽象，不建立机组启停状态。第二版原有场景及其报价、出清规则保持不变。
## 关于报价、出清与实际运行注意的地方
1. 对每个可交付的 15 分钟产品，A360 提交一张普通买单：`side = buy`，`volume_mwh = P_grid × 0.25`，`price = 3000 EUR/MWh`。零电量不提交。`price_EOM=130.92` 是优化预测值，**不是这张买单的报价**。
2. `demand_EOM` 仍按 `demand_df.csv` 的该时段数值提交 `side=buy` 的独立买单，价格使用其 `62.88 EUR/MWh`；报价超过其最高愿付价的卖单不得与之成交。继续支持旧输入未填写 `price` 时的行为。
3. `pay_as_bid` 按需求报价从高到低、供给报价从低到高撮合；仅当卖方报价不高于买方报价时成交，边际订单可部分成交。每笔成交按**对应卖单的报价**结算，同时保存撮合明细与双方费用。此机制没有全市场统一结算价，不能把 `3000` 或预测价写成实际价格。
4. 本版新增逐笔结算价；`market_results.csv` 在 `pay_as_bid` 下的原 `clearing_price_eur_per_mwh` 留空，另列示按成交量加权的 `average_trade_price_eur_per_mwh`，图表明确标为均价。调整结果数据模型以允许这个空值，不用 `0` 冒充统一出清价。原有 `pay_as_clear` 仍按第二版的统一价记录和结算。
5. 订单计划、市场成交与真实执行分开：只有成交的购电量才能供建筑运行。若 A360 买单未完全成交，先削减或取消计划中的电池充电，将成交电量优先用于热泵；满足热泵用电后的剩余成交电量，才可在功率和 SOC 约束内用于电池充电。若成交电量仍不足以满足热泵用电，则按实际供热量运行热泵并记录 `unmet_heat_mwh_th`。不得照计划更新电池 SOC 或假定未成交电量已经供给热泵；每个交付时段结束后按照实际设备运行更新真实 SOC。
6. `example_01h` 不读取 `demand_df.csv` 中的建筑负荷列。A360 在本版仅支持默认值 `P_fixed=0 MW`，相关建筑负荷列不参与优化和报价；非零固定负荷尚未验证，且不属于本版验收范围。
7. `example_01h` 不建立机组启停状态，不区分热、温、冷启动，因此不要求 `downtime_hot_start`、`downtime_warm_start` 和初始机组状态；热启动、温启动和冷启动成本不进入本场景的 EOM 报价与结算。
## 验收要求
1. `A360` 的两行正确合并为一个建筑；未知建筑列不会生成额外参与者，也不会覆盖 `A360_heat_demand=3`。
2. 热需求 `3 MW_th`、COP `2`、无蓄热时热泵耗电 `1.5 MW`，15 分钟对应 `0.375 MWh`；A360 的固定负荷使用默认值 `P_fixed=0 MW`。
3. 初始 SOC 等于最低 SOC 时电池不能放电；未提供 `max_soc` 时按 `1.0` 处理；充/放功率均不超过 `0.002 MW`，SOC 始终在配置范围内，不能同时充放，且规划窗口末端 SOC 不低于规划起点 SOC。
4. A360 买单使用 `side=buy`、非负 `volume_mwh` 和固定报价 `3000`；`demand_EOM` 买单使用 `side=buy` 和报价 `62.88`；预测价 `130.92` 不被写作任一买单的实际成交价。
5. 用两个不同报价的卖单同时满足买单时，分别按各自报价结算；A360 买单部分成交时先削减或取消电池充电并优先满足热泵，仍不足时记录未满足热需求，设备运行和 SOC 均按真实成交更新。
6. 先以短窗口验证 15 分钟调度，再以覆盖完整交付窗口的数据验证一次 96 产品开市，最后跑完整 `01h` 场景；缺失关键预测、非法 SOC 或无可用优化求解器时给出明确错误。
7. 运行第二版原有小时、统一价场景，验证其报价、结算和既有输出字段仍符合原结果。
8. 市场订单的 `volume_mwh` 统一使用非负 MWh，并通过 `side` 区分买卖方向。15 分钟产品中，`demand_EOM=100 MW` 对应 `side=buy`、`volume_mwh=25 MWh` 的买单；发电机报价功率为 `P MW` 时，对应 `side=sell`、`volume_mwh=P×0.25 MWh` 的卖单。进入供需平衡计算时买单转换为 `-25 MWh`，订单和输出文件仍保存 `25 MWh`。
9. 在单个 15 分钟产品中，当总需求低于 NEURATH F 的 `min_power=560 MW` 时，其普通卖单仍可部分成交；成交功率等于市场实际匹配的需求功率，不因 `min_power` 导致出清不可行。
10. 使用至少包含一个低价时段和一个后续高价时段的短预测序列，验证电池在功率、效率、SOC 和窗口末端 SOC 约束内低价充电、高价放电；不得使用未来实际出清价作出决策。
11. `trade_results.csv` 的每笔交易包含 `opening_id`、`buyer_bid_id` 和 `seller_offer_id`；按 `opening_id` 和 `buyer_bid_id` 汇总的 `payment_eur` 必须与 `demand_results.csv` 中对应买单的实际支付一致。

运行入口沿用 `PYTHONPATH=src python -m electricity_market_sim`，指定 `--input-dir examples/input/example_01h --scenario eom` 和独立输出目录。
