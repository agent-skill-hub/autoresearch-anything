# autoresearch-anything

> 把 Karpathy 的自动 ML 实验循环，泛化到一切可以用数字衡量的事情。你睡觉，AI 在帮你跑实验。

## 这是什么

Karpathy 的 [autoresearch](https://github.com/karpathy/autoresearch) 证明了一个强大的 pattern：给 AI agent 一份操作手册（`program.md`），让它自主跑实验循环——**改变量 -> 测结果 -> 好就留、差就扔 -> 记录学习 -> 下一轮**——循环不停，直到人类打断。

原版只做 ML 训练。但这个 pattern 的适用范围远不止于此。

**autoresearch-anything** 把这个 pattern 从 ML 实验推广到任何有明确指标的优化场景：冷邮件回复率、落地页转化率、YouTube 标题点击率、广告创意效果、产品描述转化、聊天机器人满意度、定价策略、SEO 排名……

核心思路没变：你不再手动写代码，而是写 `program.md` 来编程 AI agent 的行为。agent 自己跑，自己学，自己进化。

## 为什么重要

在自动出价的广告系统里，创意是你唯一能控制的变量。在高度竞争的市场里，每一个标题、每一封邮件、每一行文案都是一个待验证的假设。

手动 A/B 测试太慢——一周跑一个变体，一个月才跑四轮。

**autoresearch-anything 让你一晚上跑几十轮。** Agent 不睡觉，不摸鱼，不需要开会对齐。它只做一件事：不停地尝试，不停地学习。

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

## 安装

```bash
npx skills add agent-skill-hub/autoresearch-anything
```

## 怎么用

本项目提供一个 `SKILL.md` 文件，配合 Claude Code 等 AI 编程工具使用。

### 第一步：可行性检查

告诉 Claude Code 你要优化什么，它会用上面三个前提条件帮你判断场景是否可行。

### 第二步：自动改造

Claude Code 会读取 Karpathy 原版 autoresearch 仓库，理解其核心 pattern，然后根据你的场景自动生成：

- `program.md` -- 适配你场景的 agent 操作手册
- 对应的 API 交互脚本（替代原版的 `train.py`）
- 实验结果记录机制

### 第三步：启动循环

启动 agent，让它自己跑。你可以去睡觉。

```
改变量 -> 测结果 -> 好就留、差就扔 -> 记录学习 -> 下一轮
                          |                              |
                          +------------------------------+
                                  无限循环
```

每一轮的学习成果自动喂入下一轮。系统在自我进化。

## 项目结构

```
SKILL.md    -- Claude Code 技能文件，引导完成可行性检查和系统搭建
README.md   -- 你正在读的这个
```

依赖 Karpathy 的 autoresearch 仓库作为参考蓝本（SKILL.md 会自动处理克隆）。

## 部署

零基础设施要求。可以在本地跑，也可以通过 GitHub Actions 部署为定时任务，实现真正的无人值守。

## 致谢

本项目基于 [Andrej Karpathy](https://github.com/karpathy) 的 [autoresearch](https://github.com/karpathy/autoresearch)。他用一个极简的仓库证明了一个深刻的 pattern：**当你把 AI agent 的操作手册写对了，它可以自己跑下去，不需要你盯着。**

我们只是把这个 pattern 从 ML 训练推到了更多地方。

## License

MIT
