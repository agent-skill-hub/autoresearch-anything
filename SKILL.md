---
name: autoresearch-anything
description: Use when user wants to automatically and continuously optimize any metric (conversion rate, reply rate, CTR, satisfaction score, etc.) through an automated experiment loop with trace-driven causal diagnosis. Triggers - "auto research", "自动优化", "自动实验", "自我进化", "A/B循环", "自动迭代", "optimize automatically", "experiment loop", "continuous optimization"
---

# Auto Research: Automated Experiment Loop for Any Metric

## Overview

基于 Karpathy 的 AutoResearch 仓库，融合 Meta-Harness（Stanford, 2026）的 trace-driven 优化思路。

核心 pattern：写一份 `program.md` 给 AI agent 当操作手册 → agent 自己跑实验循环（改变量 → 测结果 → 记录 trace → **诊断因果** → 下一轮）→ 循环不停直到人类打断。

**与原版 autoresearch 的关键区别**（来自 Meta-Harness 论文验证）：

| 原版 | 改进版 | 为什么 |
|------|-------|-------|
| 好留差扔，git reset 丢弃失败 | results.tsv 保留所有行，标记 status 但不删 | agent 能回看失败实验，避免重蹈覆辙 |
| 只看聚合指标（如 val_bpb） | 追加 `samples.jsonl` 记录样本级结果 | 能做分群诊断（"对 A 有效但对 B 无效"） |
| 只比"本次 vs 上次" | 追加 `trace.log` 记录执行过程和假设 | agent 能分析*为什么*差，不只是*是否*差 |

**不变的**：代码变体通过 git commit 管理（不另建目录），`results.tsv` 仍是核心记录，`program.md` 仍是唯一操作手册。

仓库地址：`https://github.com/karpathy/autoresearch.git`（使用前先检查本地是否已克隆，没有则按源码下载规范克隆）

## When to Use

用户想持续自动优化某个可衡量的指标。

## When NOT to Use

- 没有客观指标（纯主观判断）
- 反馈循环太慢（> 1 周/轮）
- 没有程序化访问能力（不能通过 API 或代码修改变量、读取结果）

## Step 1: Feasibility Check

和用户确认三个前提条件。**这是判断任何场景是否适用的唯一标准，不限于下方的示例场景。**

| Prereq | What to ask | Red flag |
|--------|-------------|----------|
| Metric | 你要优化什么数字？怎么获取这个数字？ | "我看了就知道好不好" → 不够客观 |
| Variable | 你能改什么来影响这个数字？ | 实验之间没有可变因素 → 没有杠杆 |
| API | 能不能通过程序读取指标、修改变量？ | 只有手动后台 → 无法自动化（可建议用 Chrome DevTools MCP 作为替代） |

任何一项不满足就告知用户并建议替代方案。

## Step 2: Prompt Claude Code to Adapt

让 Claude Code 先读懂 autoresearch 仓库，再基于用户场景改造。Prompt 模板：

```
先阅读本地 autoresearch 仓库的 program.md、train.py、prepare.py 和 README.md，理解其自动实验循环的 pattern。

然后帮我构建一个类似的系统，但针对以下场景：

目标: [用户的场景，如"优化冷邮件回复率"]
优化指标: [如 reply_rate]（如有多个指标，指定主指标和护栏指标）
怎么获取指标: [如 Instantly API GET /api/v1/campaign/{id}/analytics]
变量: [如 email copy]
怎么修改变量: [如 Instantly API 创建新活动]
循环频率: [如每 4 小时一轮]

要求：
1. 保留 autoresearch 的核心 pattern：program.md 驱动 agent 自循环，git commit 管理变体
2. results.tsv 保留所有实验记录（包括 discard 和 crash），不删行
3. 每轮追加 samples.jsonl（样本级结果）和 trace.log（执行过程+假设）
4. agent 每轮开始前先读历史 trace 和 samples，诊断失败原因再提方案
5. 如有多个指标，在 results.tsv 中记录所有指标列，agent 关注 Pareto 权衡
```

用户如果有额外需求（部署到 GitHub Actions、添加 Slack 通知等），直接追加到 prompt 中即可。

## Step 3: File Structure

保留原版的简洁结构，只增加两个运行时产物文件：

```
program.md       — agent 操作手册（人编写）
train.py         — agent 修改的变量文件（或你场景中的等价文件）
prepare.py       — 固定的基础设施（评估、数据准备，只读）
results.tsv      — 实验记录（原版基础上：不删行 + 可扩展多指标列）
samples.jsonl    — 样本级结果（所有轮次追加写入，带 round 标记）  ← 新增
trace.log        — 执行日志 + 假设记录（所有轮次追加写入）        ← 新增
```

### results.tsv（扩展原版）

原版 5 列，按需扩展指标列。**discard 和 crash 的行保留，不删除。**

```
commit	metric_1	metric_2	status	description
a1b2c3d	0.12	0.02	keep	baseline
b2c3d4e	0.16	0.03	keep	改用正式语气标题
c3d4e5f	0.09	0.01	discard	纯 emoji 标题（回复率暴跌但退订率也降了）
d4e5f6g	0.00	0.00	crash	API key 过期
```

### samples.jsonl（新增）

每行一个样本结果，带 round 标记便于按轮次筛选。格式因场景而异：

```jsonl
{"round": "b2c3d4e", "sample_id": "email_001", "segment": "CTO", "outcome": "replied", "latency_h": 4.2}
{"round": "b2c3d4e", "sample_id": "email_002", "segment": "HR", "outcome": "no_reply"}
```

### trace.log（新增）

时间戳 + 事件，所有轮次追加写入。每轮以 HYPOTHESIS 开头，记录诊断结论和本轮计划：

```
[2026-04-03 14:00:01] === ROUND b2c3d4e ===
[2026-04-03 14:00:01] HYPOTHESIS: 上轮 c3d4e5f 对 CTO 群体回复率从 0.20 跌到 0.05，emoji 标题在专业群体不适用。本轮只改标题语气为正式，其他不动。
[2026-04-03 14:00:02] API CALL POST /campaigns {"subject": "..."}  → 201
[2026-04-03 18:00:15] API CALL GET /campaigns/camp_789/analytics → 200
[2026-04-03 18:00:16] EVAL: metric_1=0.16 (+0.07 vs c3d4e5f), metric_2=0.03 (+0.02)
[2026-04-03 18:00:17] DECISION: keep — 主指标回升，护栏指标可接受
```

## Step 4: Diagnosis Protocol

`program.md` 中必须包含以下诊断流程，agent 在每轮**提出新变体之前**执行：

```
### 每轮开始前

1. 读 results.tsv，看全局趋势（不只看上一轮）
2. 如果最近有 discard/crash 轮次，读对应的 samples.jsonl 条目，按维度拆解：
   - 哪些样本群体变好了？哪些变差了？
   - 退化是全局的还是局部的？
3. 读 trace.log 中对应轮次的执行记录，找异常：
   - API 报错或超时？
   - 变量是否真的生效了？
4. 用 git diff 对比退化轮次和成功轮次的代码差异
5. 写下本轮 HYPOTHESIS 到 trace.log（先诊断再动手）
```

**关键原则**：
- **不删历史** — results.tsv 中 discard/crash 行保留，失败和成功一样有诊断价值
- **先诊断再动手** — 不允许不看历史就盲改
- **隔离变量** — 每轮只改一个因素，避免混淆（Meta-Harness 论文最大教训：多因素同时改导致无法归因）
- **记录假设** — trace.log 中每轮必须有 HYPOTHESIS 行

## Step 5: Multi-Objective

当场景有多个指标时（常见），`program.md` 应指导 agent：

1. **定义主指标和护栏指标** — 如"主优化 reply_rate，但 unsub_rate ≤ 3%"
2. **results.tsv 记录所有指标** — 每个指标一列，Pareto 分析直接从 TSV 算
3. **agent 关注权衡** — 有时牺牲一点主指标换护栏指标大幅改善是值得的
4. **最终选择由人类决定** — agent 提供分析，人选操作点

## Scenario Quick Reference

**判断是否适用的唯一标准是 Step 1 的三个前提条件。下表只是常见场景的快速参考，任何满足前提条件的场景都适用，即使不在表中。**

| Scenario | Metric | Variable | API |
|----------|--------|----------|-----|
| Cold email | Reply rate | Email copy | Instantly / Smartlead API |
| Landing page | Conversion rate | Page content | Webflow / WordPress API |
| Ad creative | CVR / CTR | Ad text/image | Meta / Google Ads API |
| Product description | Sales / CVR | Description text | Shopify / Amazon API |
| YouTube title | CTR | Title text | YouTube Data API v3 |
| Email subject line | Open rate | Subject line | Mailchimp / SendGrid API |
| Chatbot script | CSAT score | System prompt | Custom API + survey |
| Pricing page | Signup rate | Price / copy | Stripe + analytics API |
| SEO content | Organic CTR | Meta title/desc | Google Search Console API |
| Push notification | Open rate | Message text | OneSignal / Firebase API |
