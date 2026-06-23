# weekly-stock-pick

Kevin 的三市场（A股/港股/美股）每周精选选股 Codex skill。

Built with Codex.

## 功能

- 按四因子模型筛选 A股、港股、美股候选池
- 执行催化剂、总分、板块集中度和流动性硬约束；价格/市值仅做风险提示，不做硬过滤
- 生成 self-contained HTML 周报
- 同步生成 audit 底稿，记录候选池、打分、剔除原因和催化剂来源

## 安装

复制本仓库内容到：

```bash
~/.codex/skills/weekly-stock-pick/
```

重启 Codex 后生效。

## 文件

- `SKILL.md`：skill 主流程
- `references/strategy-rules.md`：四因子和硬约束速查
- `templates/report-skeleton.html`：HTML 报告骨架
- `templates/card.html`：单张股票卡片模板
