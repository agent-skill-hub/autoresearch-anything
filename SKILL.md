---
name: autoresearch-anything
description: Use when user wants to automatically and continuously optimize any metric (conversion rate, reply rate, CTR, satisfaction score, etc.) through an automated experiment loop. Triggers - "auto research", "自动优化", "自动实验", "自我进化", "A/B循环", "自动迭代", "optimize automatically", "experiment loop", "continuous optimization"
---

# Auto Research: Automated Experiment Loop for Any Metric

## Overview

基于 Karpathy 的 AutoResearch 仓库，将其自动实验循环 pattern 改造为适用于任何场景。

核心 pattern：写一份 `program.md` 给 AI agent 当操作手册 → agent 自己跑实验循环（改变量 → 测结果 → 好就留、差就扔 → 记录 → 下一轮）→ 循环不停直到人类打断。

仓库地址：`https://github.com/karpathy/autoresearch.git`（使用前先检查本地是否已克隆，没有则按源码下载规范克隆）

仓库只有三个关键文件：
- `program.md` — 给 AI agent 的实验管理指令（核心，Karpathy 称之为 "super lightweight skill"）
- `train.py` — AI agent 修改的唯一文件（对应你场景中被优化的部分）
- `prepare.py` — 固定的基础设施（数据准备、评估，只读）
- 实验结果记录在 `results.tsv`

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
1. 保留 autoresearch 的核心 pattern：program.md 驱动 agent 自循环，改变量 → 测结果 → 好就留差就扔 → 记录 → 下一轮
2. 将 program.md 改写为适合我场景的版本
3. 将 train.py 替换为操作我的 API 的脚本
4. 保留实验结果记录机制
```

用户如果有额外需求（部署到 GitHub Actions、添加 Slack 通知、护栏指标等），直接追加到 prompt 中即可。不需要预设固定架构，让 Claude Code 基于仓库上下文自行设计。

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
