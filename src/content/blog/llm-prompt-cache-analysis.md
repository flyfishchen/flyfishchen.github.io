---
title: '大模型上下文缓存（Prompt Cache）架构与实测表现分析报告'
description: '编程 Agent 卡顿常因缓存没命中、首字太慢。结论先行：Gemini 隐式缓存不可预期、Argon 首字慢且不稳，Claude 稳定可预期；订阅用户宜 Claude Opus 做主模型、Gemini 做子代理。附三轮实测与开源子代理 skill。'
pubDate: '2026-10-20'
updatedDate: '2026-10-20'
---

## 先说结论

- **Gemini 的隐式缓存不能指望。** 官方只说"不保证节省"，没给保留时间。实测停顿几十秒到两分钟，命中就可能掉到只剩约 5k token，下一句回复要多等 5~15 秒。
- **Claude 的缓存可以预期。** 默认保留 5 分钟，每次命中自动续期。实测命中率 95% 以上，换个进程也能命中。
- **Gemini 4 Argon 首字又慢又不稳。** 受控测试中它的首字要 4~16 秒（Claude Sonnet 5.5 只要 1.5~1.8 秒），日常长会话里遇到过 10~80 秒；而且缓存命中了也没变快。
- **价格和容量已经不是差别。** 三家缓存读都是原价的 0.05~0.1 倍，上下文都在 100 万 token 左右。真正的差别是：命中有没有明确规格。
- **OpenAI 新模型也给了规格**（GPT-5.6 起承诺命中后至少保留 30 分钟），但本文没有实测。

## 怎么做

- **订阅了 Gemini Pro / Ultra**：主模型用 Claude Opus（agy 里可以直接选），Gemini 4 Argon / 3.8 Flash 当子代理，在后台读大材料、跑可以并行的任务。
- **写代码、来回改**：选缓存有明确规格的模型，比如 Claude Sonnet 5.5 / Opus 5.5。
- **一次读完大量资料、不靠缓存**：Gemini 合适，而且它能直接吃音频、视频。
- **非用 Gemini 做交互不可**：打开客户端的空闲保活（见文末附录），只能提高命中概率，不能保证。
- **把 Gemini 当子代理**：同一任务的多轮追问放在一个进程里；权限要用 hook 管。现成实现见 **[antigravity-subagent](https://github.com/flyfishchen/antigravity-subagent)**。

## 三家缓存速查

已对照官方文档核对（2026 年 10 月），来源见文末。

**Google Gemini（4 Argon / 3.8 Flash）**
- 隐式缓存默认开启，官方原话"no cost saving guarantee"，**没给保留时间**；最少 4,096 token 才可能命中。
- 显式缓存默认保留 1 小时，只能改有效期、不能追加内容，另收存储费（3.8 Flash 在 2026 年底前为 $0.50/百万 token/小时）。**Gemini 4 Argon 不支持显式缓存。**
- 缓存读：3.8 Flash 0.1 倍，4 Argon 0.05 倍。上下文约 100 万。

**Anthropic Claude（Opus 5.5 / Sonnet 5.5）**
- 显式断点或请求级自动缓存，最少 512 token。
- 默认保留 5 分钟、每次命中续期；可选 1 小时。
- 缓存读 0.05 倍；写入 1.25 倍（5 分钟）或 2 倍（1 小时），无存储费。上下文 100 万。

**OpenAI（GPT-5.6 及以后）**
- 默认开启的前缀缓存，最少 1,024 token。
- 命中后至少保留 30 分钟；更早的模型通常 5~10 分钟，可选最长 24 小时。
- 缓存读 0.1 倍（GPT-6.1 Sol 0.05 倍），写入 1.25 倍。上下文约 105 万。

三家官方都**没有**给首字延迟（TTFT）的具体数值。本文的延迟数据只来自自己的实测。

---

以下是论证过程，不关心细节可以跳过。

---

## 一、实测一：Pi + Gemini 3.8 Flash

用编程 Agent Pi 连接 `gemini-3.8-flash`，一次会话里上下文涨到 4 万~7 万 token。命中呈现严格的"二分"：连续工具调用时几乎全中，用户一停下来思考就断崖。

| 请求 | 场景 | 间隔 | 上下文 | 缓存命中 | 命中率 |
| :---: | :--- | :---: | :---: | :---: | :---: |
| 01~12 | 初始对话 | 2~13 秒 | < 19k | 0 | 0% |
| 13~16 | 连续工具调用 | 2~12 秒 | 40k~49k | 18,309 → 41,990 | 85%~92% |
| **17** | **等用户输入** | **81.7 秒** | 49,827 | **5,357** | **~10%** |
| 18 | 工具返回 | 8.1 秒 | 51,967 | 47,212 | 90.8% |
| **19** | **等用户输入** | **52.3 秒** | 52,701 | **5,369** | **~10%** |
| 20~23 | 连续工具调用 | 4~12 秒 | 54k~62k | 50,054 → 56,240 | 90%+ |
| **24** | **等用户输入** | **98.5 秒** | 63,077 | **5,439** | **~8%** |
| 25~31 | 连续工具调用 | 3~8 秒 | 64k~73k | 60,407 → 70,451 | 96%+ |

体感：命中时首字不到 1 秒；断崖那几次光标要停 5~15 秒。

## 二、实测二：用 Antigravity CLI 复测

把 Google Antigravity CLI（`agy` 1.4.2）当作 Claude Code 的子代理，以无头模式运行，从每一步返回的用量里读取命中数。任务是读写几个小文件，上下文约 7k~17k token。

**同一进程连续 3 轮（间隔 2~10 秒）**

| 模型 | 首步命中 | 后续命中 | 结论 |
| :--- | :---: | :---: | :--- |
| Gemini 3.8 Flash | 0 | 0 | 一次没中 |
| Gemini 4 Argon | 0 | 5,362~5,739 | 只中约 5.3k |
| Claude Sonnet 5.5 | 0 | 11,243~12,708 | 95% 以上 |

**跨进程**
- **Claude**：全新进程第一步就命中 40,365（前几分钟别的调用写入了同样的前缀）；续接会话命中 64,920。
- **Gemini 3.8 Flash**：从没命中。
- **Gemini 4 Argon**：约 2 分钟后续接命中 72%，同模型单进程两轮合计只有 54%，波动很大。

（经 agy 调用的 Claude 走 Google 的通道，这里只比较命中，不比较价格。）

## 三、实测三：首字延迟

同样用 agy 无头运行，逐行给输出事件打时间戳。下表已扣掉 agy 自身启动的 5~7 秒，只算模型本身。问题是一道不需要工具的简答题（约 7k token 上下文）；大上下文组粘贴了约 6 万 token 源码，并在同一会话里追问一轮。

| 模型 / 思考等级 | 次数 | 首字延迟 |
| :--- | :---: | :---: |
| Gemini 4 Argon / medium | 5 | 4.0~6.5 秒 |
| Gemini 4 Argon / low | 3 | 4.8~16.5 秒 |
| Gemini 3.8 Flash / medium | 3 | 3.6~4.7 秒 |
| Claude Sonnet 5.5 / medium | 3 | 1.5~1.8 秒 |
| Argon / medium，6 万 token，首轮 | 2 | 约 7 秒 |
| Argon / medium，6 万 token，追问（缓存命中 96%） | 2 | 7.0~9.6 秒 |

- **Argon 最慢，而且波动大。** 调到 low 也没变快，其中一次要 16.5 秒。
- **缓存命中没有缩短 Argon 的首字时间。** 6 万 token 命中 96% 时反而略慢，说明它的等待主要不在重新预填充（推测是服务端排队或隐藏的思考：agy 报告 Argon 的思考 token 一直为 0，而 Flash 能报出 225~483）。
- **受控测试没有复现 10~80 秒。** 日常长会话里的极端值，可能和更长的上下文、连续工具调用或高峰时段有关（推测）。本组样本少、只测了一个时段，仅供参考。

## 四、为什么会这样

### 1. 隐式缓存本来就不保证
Google 文档把隐式缓存定性为"不保证节省"，只建议"在短时间内发送前缀相似的请求"，没有给出保留时间。Pi 实测里停顿 52 秒以上就掉；agy 复测里 Argon 隔 2 分钟还能中 72%。所以"30~60 秒"只是经验区间，不是规格。

### 2. 门槛之上也可能不中
Gemini 3.8 Flash 的官方门槛是 4,096 token。复测时提示词有 7k~8k，间隔只有 2~3 秒，仍然一次没中。这和 Pi 实测里前 12 个请求全是 0 一致：只满足"超过门槛、间隔短"还不够。

### 3. 约 5.36k 的"保底命中"从哪来（推测）
Pi 每次断崖后都精确停在 5,357~5,488。初版的解释是"这是 Pi 的系统提示加工具定义，作为全网共享的热点前缀常驻缓存"。

复测推翻了这个解释：agy 的系统提示完全不同，固定前缀约 7.4k，保底却同样是 5,362。两个不同的 Agent 得到几乎同一个数，更可能是平台侧的因素，比如缓存的分块粒度，或服务端注入的公共前缀。具体机制待验证。

### 4. 断崖为什么这么慢
命中时只需计算新增的少量 token；不中时要把 5 万~7 万 token 全部重新预填充一遍，所以首字等待从不到 1 秒变成 5~15 秒。连接空闲关闭后请求被调度到没有该会话缓存的机器上，也可能是原因之一（推测）。

### 5. 为什么 Claude 稳定
Claude 的缓存有明确规格：5 分钟有效、每次命中续期、前缀字节一致就能复用。实测中跨进程、跨会话都能命中，与规格一致。

## 五、把 agy 接成子代理时的权限坑

- **子代理绕过了主代理的审批**：Claude Code 只看到"执行了一条命令"，agy 之后读写了什么、跑了什么都不经过审批。
- **测试机上 agy 不受任何限制**：设置为 `always-proceed`，可以写工作区外的文件、访问外网。
- **Ubuntu 上沙箱默认不可用**：AppArmor 默认禁止非特权 user namespace，`--sandbox` 既不报错也不生效。
- **能管住它的只有 PreToolUse hook**，但有几个细节：
  - 只有 `~/.gemini/config/hooks.json` 和工作区下的 `.agents/hooks.json` 会执行，`~/.gemini/antigravity-cli/hooks.json` 加载了也不执行；
  - hook 输出为空、或超时，都按拒绝处理；
  - 被拒后 agy 常换别的工具绕过。

skill 里因此做了三档策略（只读 / 工作区 / 不限），被拒的操作分为硬拒绝、待用户审批、交还主代理三类。代码和测试见 **[github.com/flyfishchen/antigravity-subagent](https://github.com/flyfishchen/antigravity-subagent)**。

## 六、主模型和子代理怎么分工

对订阅了 Gemini Pro / Ultra、在 agy 里同时能用 Claude 和 Gemini 的人，上面的数据指向同一个分工：

- **主模型用 Claude Opus。** 主模型要一直陪你对话，每句之间你会停下来想，Gemini 正好在这时掉缓存、首字又慢；Claude 的缓存命中稳定，首字快。
- **Argon / 3.8 Flash 当子代理。** 子代理在后台跑、可以并行，首字慢几秒影响不大。最适合"一次塞进大量材料、出一次结论"的活，比如读整个仓库、长日志、长文档。这类任务本来就不靠缓存。
- **按额度分配。** 订阅制下花的是额度。Claude 额度更宝贵，留给主循环；吃 token 的阅读任务交给 Gemini。
- **别派零碎小活。** agy 每次请求固定带约 7.4k token 前缀，进程启动要 3~10 秒；一个只统计文件数的小任务，它自己又开了子代理，花了 5.8 万 token。
- **结果要核对。** 子代理的结论只是线索，主模型要对照它实际做了什么再采信。

## 附录：Pi 的缓存保活配置

Pi 没有接 Google 的显式缓存（显式缓存建好后不能追加内容，Gemini 4 Argon 也根本不支持），但提供了空闲保活。以下配置以 Pi 自己的文档为准：

1. `~/.pi/agent/settings.json`：
   ```json
   { "cacheWarming": "idle" }
   ```
2. `~/.pi/agent/models.json` 中对应模型：
   ```json
   {
     "id": "gemini-3.8-flash",
     "promptCache": { "short": 30, "long": 300 }
   }
   ```

用户停顿约 25 秒时，Pi 会在后台发一个极小的请求来"碰"一下缓存。按第四节的结论，这能提高命中概率，但不能保证命中。

## 参考资料

- Google：[Context caching](https://ai.google.dev/gemini-api/docs/generate-content/caching)、[Pricing](https://ai.google.dev/gemini-api/docs/pricing)、[Gemini 3.8 Flash](https://ai.google.dev/gemini-api/docs/models/gemini-3.8-flash)
- Anthropic：[Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- OpenAI：[Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)、[Models](https://developers.openai.com/api/docs/models)
- 实测数据与子代理实现：[flyfishchen/antigravity-subagent](https://github.com/flyfishchen/antigravity-subagent)
