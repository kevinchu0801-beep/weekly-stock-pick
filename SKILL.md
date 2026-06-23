---
name: weekly-stock-pick
description: Kevin 的三市场（A股/港股/美股）每周精选选股工作流。按四因子打分模型（动量30+资金30+催化剂20+估值20）和硬约束（催化剂≥8、总分≥50、同板块≤2、流动性达标；不按价格/市值带剔除）扫描候选池，每市场取前 5 名（宁缺毋滥），输出含买方分析师 11 模块精简 5 段卡片的 self-contained HTML 报告到 reports/YYYY-WXX.html。当用户要求"每周选股 / 周一选股报告 / 三市场精选 / W23 报告 / 重新选一次股票 / 摒弃之前结论再选股"等请求时触发。
description_zh: 每周三市场选股报告
description_en: Weekly tri-market stock pick
disable: false
agent_created: true
---

# weekly-stock-pick

## When to use

Trigger this skill when the user asks for any of the following:
- "每周选股 / 周一选股报告 / 帮我选股"
- "三市场精选 / A股港股美股选股"
- "W23 报告 / 这周的选股"
- "重新选一次 / 摒弃之前结论再选股"
- automation 自动注入 `/Users/liali/WorkBuddy/选股策略/prompts/weekly-pick.md` 的 prompt
- 用户提到 `STRATEGY.md` 四因子或要求按四因子打分扫描股票

Do NOT trigger for: 单只股票深度尽调（用 trading-analysis）、纯行情查询（用 finance-data 数据源）、日内交易信号。

## Steps

### Step 0 · 环境检查
执行前先确认以下路径存在；缺失时先说明缺项并停止，不要半路生成残缺报告：
- `/Users/liali/WorkBuddy/选股策略/STRATEGY.md`
- `/Users/liali/WorkBuddy/选股策略/prompts/weekly-pick.md`
- `/Users/liali/.workbuddy/plugins/marketplaces/cb_teams_marketplace/plugins/finance-data/skills/westock-data/scripts/index.js`
- `/Users/liali/WorkBuddy/选股策略/reports/`
- `/Users/liali/WorkBuddy/日常问题/.workbuddy/memory/`

### Step 1 · 读策略与时间锚
1. Read `/Users/liali/WorkBuddy/选股策略/STRATEGY.md` to confirm latest factor weights/hard constraints (Kevin 可能临时调整权重)。
2. Read `/Users/liali/WorkBuddy/选股策略/prompts/weekly-pick.md` to confirm pipeline.
3. 确认当前 ISO 周编号（用 `date +"%Y-W%V"`）和报告文件名 `reports/YYYY-WXX.html`。

### Step 2 · 拉候选池（westock-data）
westock-data 调用方式（CLI 不在 PATH 时直接 node 调用）：
```bash
WD="/Users/liali/.workbuddy/binaries/node/versions/22.22.2/bin/node /Users/liali/.workbuddy/plugins/marketplaces/cb_teams_marketplace/plugins/finance-data/skills/westock-data/scripts/index.js"
```

候选池采集（每市场 30-50 只，优先按 `weekly-pick.md` 的完整来源执行）：
```bash
$WD hot stock --market hs --limit 30            # A股热搜
$WD sector --rank interval_chg_rank_sw1 --sort chg20Days --limit 10  # A股20日最强板块
$WD lhb                                         # 近一周龙虎榜机构榜，筛机构净买入>5000万
$WD reserve sh000001                            # 业绩预告超预期候选

$WD hot stock --market hk --limit 30            # 港股热搜
$WD marketnews hk                               # 港股近一周大事件涉及股

$WD hot stock --market us --limit 30            # 美股热搜
$WD marketnews us                               # 美股近一周大事件涉及股
```

从最强板块取 5 只龙头，与热搜、龙虎榜/业绩预告/marketnews 并集去重；再与投资池范围相交，组成候选池。若某个命令不可用，记录缺失来源并用其他来源补足，不要伪造数据。

### Step 3 · 批量拉数据
```bash
# 现价/PE/PB/市值/涨幅（一次最多 15-20 只）
$WD quote sh600522,sh600487,sz000988,...
$WD quote hk02382,hk01024,hk02018,...
$WD quote usAMD,usMRVL,usNOW,usCRWD,...

# K线/催化剂/新闻（用于动量和未来30-60天事件）
$WD kline --period day --limit 60 <代码>
$WD calendar <代码>
$WD news <代码>

# 资金流（按市场分别拉）
$WD asfund <a股代码,逗号分隔>     # 主力净流入
$WD hkfund <港股代码,逗号分隔>    # 南向资金
$WD usfund <美股代码,逗号分隔>    # 卖空数据

# 机构信号（10 只一批）
$WD rating <代码,逗号分隔>        # 评级上调/下调
$WD consensus <a股代码,逗号分隔>  # 一致预期上修（港股可能无数据，跳过）
```

如果 `calendar/news` 无法确认未来 30-60 天催化剂，每只股票最多做 1 次 WebSearch 交叉验证，并在数据底稿中记录来源；仍无法确认则因子3不得硬凑，按无催化剂处理。

### Step 4 · 硬约束过滤（必须严格执行）
1. **不按价格/市值带剔除**：股价、市值只记录在底稿和卡片中，用于风险提示、拥挤度判断和仓位讨论；不得因市值过大/过小或股价高低直接踢出
2. **催化剂因子**：因子3 ≥ 8 分；无未来 30-60 天具体催化的票直接踢
3. **总分**：≥ 50 分
4. **同板块**：每市场同板块最多 2 只，按总分高低保留
5. **流动性**：A股 20 日均额 ≥ 5000 万 RMB；港股 ≥ 1 亿 HKD；美股 ≥ 5000 万 USD

如果某市场合格票 < 5 只 → **宁缺毋滥**，写"本周仅 N 只入选"，不要凑数。

### Step 5 · 四因子打分
按 `STRATEGY.md` 第二节计算四因子总分：
- 因子1 动量（30）：60 日相对涨幅 + 20 日量价 + MA60 站上/下
- 因子2 资金（30）：20 日主力净流入 + 北向/机构持仓 + 30 日评级上调净数
- 因子3 催化剂（20）：30 天财报 + 60 天产品/政策 + 板块β
- 因子4 估值（20）：PE/PB 5 年分位 + 营收增速 + 一致预期上修

总分 → 分类：
- 总分≥80 + 4 因子≥10 → **确信** · 5%
- 总分 65-80 + 因子3≥12 → **入场** · 3%
- 总分≥50 但因子4<10 → **交易** · 1% 试仓
- 入榜 Top 5 但临界 → **观察池**

### Step 6 · 写卡片（每张 ≤300 字，5 段必含）
每张卡片包含：
1. **一句话生意 + 护城河**（≤40 字，小学生能懂）
2. **核心催化（30-60 天）**：2-3 条具体事件 + 日期
3. **共识 vs 反共识**：市场怎么想 / 我们怎么想
4. **风险点**：2 条，含明确失效条件（"跌破 X 元 → 趋势破坏"）
5. **二元结论**：分类 + 仓位 + 时间窗 + 止损 + "成功如果___ / 失败如果___"

**绝对不要做**：
- 不给目标价（你不是卖方）
- 不复述新闻（催化剂必须是"未来"，不是"上周")
- 不用 quant 黑话（Kevin 是产品经理）
- 不凑数（合格<5 直接标注"仅 N 只"）

### Step 7 · 渲染 HTML
使用 `@templates/report-skeleton.html` 作为骨架、`@templates/card.html` 作为单张卡片模板（浅色主题：背景 #f5f6f8、卡片 #ffffff、主文字 #1f2329、深蓝 #2f6fed + 深橙 #d97706），填入数据后输出到：

```
/Users/liali/WorkBuddy/选股策略/reports/YYYY-WXX.html
```

**禁止**在文件名加 `-v1`/`-v2`/`-修正版` 等版本标识 —— 同周覆盖写入。

同时输出可复盘数据底稿：

```
/Users/liali/WorkBuddy/选股策略/reports/YYYY-WXX-audit.md
```

底稿必须包含：候选池来源、每只股票原始指标、四因子分、硬约束剔除原因、最终排序、催化剂来源；若市值或股价极端，只记录为风险提示，不作为剔除原因。

### Step 8 · 收尾（强制）
1. 确认报告路径，并在 Codex 中给出本地文件链接；如果可用，用 Browser 打开 `file:///Users/liali/WorkBuddy/选股策略/reports/YYYY-WXX.html` 预览。
2. 在 `/Users/liali/WorkBuddy/日常问题/.workbuddy/memory/YYYY-MM-DD.md` 追加"本周精选已生成 + 入选清单 + 关键判断"
3. 在选股策略项目的 daily memory 也记一笔
4. 更新 `/Users/liali/WorkBuddy/选股策略/reports/index.html`，加入本周报告链接
5. 最终交付 HTML 路径和 audit 底稿路径

## Pitfalls

- **westock-data 不在 PATH**：必须用绝对路径 node 调用 `/Users/liali/.workbuddy/plugins/marketplaces/cb_teams_marketplace/plugins/finance-data/skills/westock-data/scripts/index.js`
- **港股 consensus 缺失**：`consensus hk*` 大概率返回 `[SKILL_004] 未找到一致预期数据`；港股估值因子改用 `quote` 中 PE/PB
- **市值/股价口径**：市值和股价只做展示与风险提示，不做硬过滤；港股市值如需跨市场比较，可用 HKD÷1.1 近似换算 RMB
- **同板块挤出**：半导体、AI 算力、网络安全很容易超 2 只；按总分排序后只保留前 2，被挤出的放观察池
- **HTML 写入截断**：单次 `Write` 写超 50KB 容易截断，建议分段写或先写骨架再 `Edit` 追加每张卡片
- **市值单位混淆**：A 股 quote 返回的市值是亿元，美股是亿美元；不要相互比较时混算
- **不要假装数据完整**：某只票数据缺失就直接跳过 + 候选池补一个，不要硬填 0 分混进结果
- **催化剂来源不足**：未来事件必须有 `calendar/news` 或 WebSearch 来源；只有历史新闻不能算催化剂
- **模板注释泄漏**：最终报告不得保留卡片说明注释；单卡片按 `@templates/card.html` 生成

## Verification

报告交付前自检 6 项：
1. ☑ HTML 文件以 `</html>` 闭合，可用 `tail -5 reports/YYYY-WXX.html` 验证
2. ☑ 三市场每张卡片都含 5 段必备模块（一句话生意 / 催化 / 共识反共识 / 风险 / 二元结论）
3. ☑ 总仓位 ≤ 35%（确信 5% × N + 入场 3% × N + 交易 1% × N），保留 ≥ 65% 现金
4. ☑ 没有出现"目标价"三个字；没有 v1/v2/修正版后缀
5. ☑ `YYYY-WXX-audit.md` 已写入，并列出每只被剔除股票的原因；剔除原因不得是价格带/市值带
6. ☑ `reports/index.html` 已更新本周入口

## References

- `@references/strategy-rules.md` — 四因子打分细则（从 STRATEGY.md 抽取的核心规则）
- `@templates/report-skeleton.html` — HTML 报告骨架（浅色主题 + 卡片样式）
- `@templates/card.html` — 单张股票卡片模板
