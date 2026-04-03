# autoresearch-anything

> 把 Karpathy 的自动 ML 实验循环，泛化到一切可以用数字衡量的事情。你睡觉，AI 在帮你跑实验。

## 这是什么

Karpathy 的 [autoresearch](https://github.com/karpathy/autoresearch) 证明了一个强大的 pattern：给 AI agent 一份操作手册（`program.md`），让它自主跑实验循环——**改变量 -> 测结果 -> 记录学习 -> 下一轮**——循环不停，直到人类打断。

**autoresearch-anything** 把这个 pattern 从 ML 训练推广到任何有明确指标的优化场景，并融合了 [Meta-Harness](https://arxiv.org/abs/2603.28052)（Stanford, 2026）的三个关键改进：

| 原版 autoresearch | 改进版 |
|-------------------|-------|
| 好留差扔，只保留最优 | **全量保留** — 所有候选的代码+指标+trace 都留存 |
| 只看聚合指标（如回复率 12%） | **样本级记录** — 每封邮件/每个用户的结果都记录 |
| 只比"好 vs 差" | **trace 诊断** — agent 读执行日志，分析*为什么*差，再提改进方案 |

Meta-Harness 论文的消融实验证明：只看分数的优化准确率 34.6%，加上 trace 诊断后跳到 50.0%。这个差距就是"知道结果"和"理解原因"的区别。

## 为什么重要

手动 A/B 测试太慢——一周跑一个变体，一个月才跑四轮。

**autoresearch-anything 让你一晚上跑几十轮。** Agent 不睡觉，不摸鱼，不需要开会对齐。而且它不是盲目试错——每轮开始前，它会回顾所有历史实验的 trace，诊断失败原因，隔离变量，再有针对性地提方案。

## 三个前提条件

在开始之前，你的场景必须同时满足以下三项。缺一不可。

| 条件 | 问自己 | 不满足的信号 |
|------|--------|------------|
| **客观指标** | 要优化什么数字？怎么获取？ | "我看了就知道好不好" -- 主观判断无法自动化 |
| **可调变量** | 能改什么来影响这个数字？ | 实验之间没有可变因素 -- 没有优化杠杆 |
| **API 访问** | 能通过程序读取指标、修改变量？ | 只有手动后台 -- 无法形成自动循环 |

三个条件全部满足？往下看。

## 适用场景

以下只是常见示例。**任何满足三个前提条件的场景都适用。**

| 场景 | 优化指标 | 可调变量 | API |
|------|---------|---------|-----|
| 冷邮件 | 回复率 | 邮件正文 | Instantly / Smartlead |
| 落地页 | 转化率 | 页面内容 | Webflow / WordPress |
| 广告创意 | CVR / CTR | 文案 / 素材 | Meta / Google Ads |
| 产品描述 | 销量 / 转化率 | 描述文本 | Shopify / Amazon |
| YouTube 标题 | 点击率 | 标题文字 | YouTube Data API |
| 邮件标题 | 打开率 | Subject line | Mailchimp / SendGrid |
| 聊天机器人 | 满意度评分 | System prompt | 自定义 API |
| 定价页面 | 注册率 | 价格 / 文案 | Stripe + 分析工具 |
| SEO 内容 | 自然搜索 CTR | Meta title / desc | Google Search Console |
| 推送通知 | 打开率 | 消息文本 | OneSignal / Firebase |

## 怎么用

本项目提供一个 `SKILL.md` 文件，配合 Claude Code 等 AI 编程工具使用。

### 第一步：可行性检查

告诉 Claude Code 你要优化什么，它会用上面三个前提条件帮你判断场景是否可行。

### 第二步：自动改造

Claude Code 会读取 Karpathy 原版 autoresearch 仓库，理解其核心 pattern，然后根据你的场景自动生成：

- `program.md` -- 适配你场景的 agent 操作手册（含诊断协议和 trace 记录要求）
- 对应的 API 交互脚本（替代原版的 `train.py`）
- 实验文件系统结构（每轮一个目录：变量代码 + 聚合指标 + 样本级结果 + 执行 trace）

### 第三步：启动循环

启动 agent，让它自己跑。你可以去睡觉。

```
读历史 trace → 诊断 → 提假设 → 改变量 → 测结果 → 记录(代码+指标+样本+trace) → 下一轮
    |                                                                              |
    +------------------------------------------------------------------------------+
                                      无限循环
```

每一轮的完整记录自动喂入下一轮诊断。系统在自我进化，而且知道*为什么*进化。

## 实验文件系统

```
experiments/
├── overview.json          # Pareto 前沿 + 全局排行
├── round_001/
│   ├── variant.py         # 本轮变量代码
│   ├── scores.json        # 聚合指标
│   ├── samples.jsonl      # 样本级结果
│   └── trace.log          # 完整执行日志 + 假设记录
├── round_002/
│   └── ...
└── ...
```

## 项目结构

```
SKILL.md    -- Claude Code 技能文件，引导完成可行性检查和系统搭建
README.md   -- 你正在读的这个
```

依赖 Karpathy 的 autoresearch 仓库作为参考蓝本（SKILL.md 会自动处理克隆）。

## 部署

零基础设施要求。可以在本地跑，也可以通过 GitHub Actions 部署为定时任务，实现真正的无人值守。

## 致谢

- [Andrej Karpathy](https://github.com/karpathy) 的 [autoresearch](https://github.com/karpathy/autoresearch) — 证明了"用操作手册编程 AI agent"的 pattern
- [Meta-Harness](https://arxiv.org/abs/2603.28052)（Stanford, 2026）— 证明了 trace-driven 优化远优于 score-only 优化

## License

MIT
