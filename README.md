# autoresearch-anything

> 把 Karpathy 的自动 ML 实验循环，泛化到一切可以用数字衡量的事情。你睡觉，AI 在帮你跑实验。

## 这是什么

Karpathy 的 [autoresearch](https://github.com/karpathy/autoresearch) 证明了一个强大的 pattern：给 AI agent 一份操作手册（`program.md`），让它自主跑实验循环——**改变量 -> 测结果 -> 记录 -> 下一轮**——循环不停，直到人类打断。

**autoresearch-anything** 把这个 pattern 从 ML 训练推广到任何有明确指标的优化场景，并融合了 [Meta-Harness](https://arxiv.org/abs/2603.28052)（Stanford, 2026）的三个改进：

| 原版 autoresearch | 改进版 |
|-------------------|-------|
| 好留差扔，git reset 丢弃失败 | **不删历史** — results.tsv 保留所有行，标记 status 但不删 |
| 只看聚合指标 | **样本级记录** — 追加 samples.jsonl，能做分群诊断 |
| 只比"本次 vs 上次" | **trace 诊断** — 追加 trace.log，agent 分析*为什么*差再动手 |

**不变的**：代码变体通过 git commit 管理，results.tsv 仍是核心记录，program.md 仍是唯一操作手册。不另建目录体系，不重复 git 已有的版本管理。

Meta-Harness 论文的消融实验证明：只看分数的优化准确率 34.6%，加上 trace 诊断后跳到 50.0%。这个差距就是"知道结果"和"理解原因"的区别。

## 三个前提条件

| 条件 | 问自己 | 不满足的信号 |
|------|--------|------------|
| **客观指标** | 要优化什么数字？怎么获取？ | "我看了就知道好不好" — 主观判断无法自动化 |
| **可调变量** | 能改什么来影响这个数字？ | 实验之间没有可变因素 — 没有优化杠杆 |
| **API 访问** | 能通过程序读取指标、修改变量？ | 只有手动后台 — 无法形成自动循环 |

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

## 文件结构

保留原版的简洁性，只增加两个运行时产物文件：

```
program.md       — agent 操作手册（人编写）
train.py         — agent 修改的变量文件（或你场景中的等价文件）
prepare.py       — 固定的基础设施（评估、数据准备，只读）
results.tsv      — 实验记录（不删行 + 可扩展多指标列）
samples.jsonl    — 样本级结果（所有轮次追加写入）  ← 新增
trace.log        — 执行日志 + 假设记录              ← 新增
```

## 怎么用

### 第一步：可行性检查

告诉 Claude Code 你要优化什么，它会用三个前提条件帮你判断场景是否可行。

### 第二步：自动改造

Claude Code 读取 Karpathy 原版 autoresearch 仓库，理解核心 pattern，根据你的场景自动生成适配的 program.md 和脚本。

### 第三步：启动循环

```
读 results.tsv + trace.log → 诊断 → 写 HYPOTHESIS → 改变量 → git commit → 测结果 → 记录 → 下一轮
    |                                                                                            |
    +--------------------------------------------------------------------------------------------+
                                              无限循环
```

每一轮的完整记录喂入下一轮诊断。agent 不只知道好坏，还知道*为什么*好坏。

## 部署

零基础设施要求。可以在本地跑，也可以通过 GitHub Actions 部署为定时任务，实现真正的无人值守。

## 致谢

- [Andrej Karpathy](https://github.com/karpathy) 的 [autoresearch](https://github.com/karpathy/autoresearch) — 证明了"用操作手册编程 AI agent"的 pattern
- [Meta-Harness](https://arxiv.org/abs/2603.28052)（Stanford, 2026）— 证明了 trace-driven 优化远优于 score-only 优化

## License

MIT
