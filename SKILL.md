---
name: autoresearch-anything
description: Use when user wants to automatically and continuously optimize any metric (conversion rate, reply rate, CTR, satisfaction score, etc.) through an automated experiment loop with full trace history and causal diagnosis. Triggers - "auto research", "自动优化", "自动实验", "自我进化", "A/B循环", "自动迭代", "optimize automatically", "experiment loop", "continuous optimization"
---

# Auto Research: Automated Experiment Loop for Any Metric

## Overview

基于 Karpathy 的 AutoResearch 仓库，融合 Meta-Harness（Stanford, 2026）的 trace-driven 优化思路。

核心 pattern：写一份 `program.md` 给 AI agent 当操作手册 → agent 自己跑实验循环（改变量 → 测结果 → **全量保留历史** → **分析 trace 诊断因果** → 下一轮）→ 循环不停直到人类打断。

**与原版 autoresearch 的关键区别**（来自 Meta-Harness 论文验证的三个改进）：

| 原版 | 改进版 |
|------|-------|
| 好留差扔，只保留最优 | **全量保留**，所有候选的代码+指标+trace 都留存 |
| 只看聚合指标（如回复率 12%） | **样本级记录**（每封邮件/每个用户/每个样本的结果） |
| 只比"好 vs 差" | **trace 诊断**（agent 读执行日志，分析*为什么*差） |

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
优化指标: [如 reply_rate]
怎么获取指标: [如 Instantly API GET /api/v1/campaign/{id}/analytics]
变量: [如 email copy]
怎么修改变量: [如 Instantly API 创建新活动]
循环频率: [如每 4 小时一轮]

要求：
1. 保留 autoresearch 的核心 pattern：program.md 驱动 agent 自循环
2. 采用下方"实验文件系统"结构，全量保留历史
3. 每轮评估记录样本级结果和执行 trace
4. agent 每轮开始前先读历史 trace，诊断失败原因再提方案
```

用户如果有额外需求（部署到 GitHub Actions、添加 Slack 通知、护栏指标等），直接追加到 prompt 中即可。不需要预设固定架构，让 Claude Code 基于仓库上下文自行设计。

## Step 3: Experiment Filesystem（核心改进）

每轮实验不再只写一行 TSV，而是生成一个完整目录。以下结构是 `program.md` 必须要求 agent 遵守的：

```
experiments/
├── overview.json          # Pareto 前沿 + 全局排行（agent 每轮更新）
├── round_001/
│   ├── variant.py         # 本轮使用的变量代码（或 variant.json / variant.txt）
│   ├── scores.json        # 聚合指标 {"reply_rate": 0.12, "unsub_rate": 0.02, ...}
│   ├── samples.jsonl      # 样本级结果，每行一个样本
│   └── trace.log          # 完整执行日志（API 调用、返回值、报错、耗时）
├── round_002/
│   ├── variant.py
│   ├── scores.json
│   ├── samples.jsonl
│   └── trace.log
└── ...
```

### 各文件规范

**overview.json** — agent 每轮结束后更新，记录：
- `pareto_frontier`: 多维度非劣解列表（如 [回复率, 退订率] 的 Pareto 前沿）
- `best_by_metric`: 每个指标的历史最优轮次
- `total_rounds`: 已完成轮数

**scores.json** — 本轮聚合指标，所有指标用 key-value 记录：
```json
{"reply_rate": 0.12, "unsub_rate": 0.02, "cost_per_reply": 1.35}
```

**samples.jsonl** — 每行一个样本的完整结果，格式因场景而异：
```jsonl
{"sample_id": "email_001", "recipient_segment": "CTO", "outcome": "replied", "latency_hours": 4.2}
{"sample_id": "email_002", "recipient_segment": "HR", "outcome": "no_reply", "latency_hours": null}
```

**trace.log** — 时间戳 + 事件的执行日志：
```
[2026-04-03 14:00:01] START round_003
[2026-04-03 14:00:02] API CALL POST /campaigns {"subject": "..."}  → 201 {"id": "camp_789"}
[2026-04-03 14:00:03] HYPOTHESIS: 上轮对 CTO 群体回复率下降，疑似标题过于 casual，本轮改用正式语气
[2026-04-03 18:00:15] API CALL GET /campaigns/camp_789/analytics → 200 {"sent": 50, "replied": 8}
[2026-04-03 18:00:16] EVAL: reply_rate=0.16 (+0.04 vs round_002), unsub_rate=0.01 (-0.01)
[2026-04-03 18:00:17] END round_003
```

## Step 4: Agent Diagnosis Protocol（核心改进）

`program.md` 中必须包含以下诊断流程，在每轮**提出新变体之前**执行：

```
### 每轮开始前的诊断步骤

1. 读 overview.json，了解当前 Pareto 前沿和历史最优
2. 读最近 3 轮（或全部退化轮次）的 samples.jsonl，按维度拆解：
   - 哪些样本群体变好了？哪些变差了？
   - 退化是全局的还是局部的？
3. 读对应轮次的 trace.log，找执行异常：
   - API 报错或超时？
   - 变量是否真的生效了？（防止"以为改了但其实没改"）
4. 对比退化轮次和成功轮次的 variant 代码差异
5. 写下本轮假设：
   - 基于以上诊断，你认为问题出在哪？
   - 本轮打算改什么？预期效果是什么？
6. 将假设记录到本轮的 trace.log 开头（HYPOTHESIS 行）
```

**关键原则**：
- **全量保留，不丢弃任何轮次** — 失败的实验和成功的一样有价值
- **先诊断再动手** — 不允许不看历史就盲改
- **隔离变量** — 每轮只改一个因素，避免混淆（Meta-Harness 论文里最大的教训）
- **记录假设** — 每轮 trace 开头必须写明"我认为问题是 X，所以改 Y"

## Step 5: Multi-Objective & Pareto

当场景有多个指标时（常见），`program.md` 应指导 agent：

1. **定义主指标和护栏指标** — 如"主优化回复率，但退订率不得超过 3%"
2. **维护 Pareto 前沿** — overview.json 中记录所有非劣解，不只是单维最优
3. **agent 可以沿前沿不同方向探索** — 有时牺牲一点主指标换护栏指标大幅改善是值得的
4. **最终选择由人类决定** — agent 提供前沿，人选操作点

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
