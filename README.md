# Do Not Safety Me

**让模型生成的坏草稿死在历史之外。**

> **Reject before history. Retry from a clean context.**

模型生成了一段文字，不代表这段文字已经应该成为“对话历史”。

`Do Not Safety Me` 是一份轻量的 **LLM output admission / isolation** 教程：当模型在一个本来正常的对话里突然生成不合适的升级、模板化拒答、协议泄漏或其他不应进入会话事实的内容时，应用层先把它当作 **candidate**，在持久化和发送之前做本地检查；若拒绝，则隔离原始 draft、清理可能被污染的 provider context，并且最多从干净上下文重试一次。

最小思路只有一句：

```text
generation != commitment
```

也就是：

```text
model output
   ↓
candidate
   ↓
local admission
   ├─ PASS   → persist → history → memory → delivery
   └─ REJECT → metadata-only receipt
                 ↓
             clean context
                 ↓
             retry once
               ├─ PASS   → persist
               └─ REJECT → fail closed
```

这套模式来自一个长期运行的 AI companion 项目里真实踩过的坑。公开版本只保留通用工程经验与合成测试，不包含私人 prompt、真实对话或私有 detector 规则。

---

## 先说清楚：这个项目不是什么

仓库名故意有点坏，但这里讲的**不是关闭 provider safety、不是绕过模型平台的安全策略，也不是把真实紧急情况过滤掉**。

它解决的是 host application 自己拥有的一个更基础的问题：

> **模型输出什么时候才算这个应用真的说过？**

一个 provider 可以成功返回 HTTP 200，一段 completion 也可以语法完全正常，但你的应用仍然可能有理由拒绝把它提交为 canonical assistant message。

例如：

- 情感 / 假设性对话突然被模型错误升级成现实应急流程；
- 角色扮演中突然出现模板化 meta-refusal；
- 模型把内部协议、delimiter 或控制标记写进正文；
- 一个工具型 Agent 声称“已经调用工具”，但 host ledger 没有对应调用；
- 输出违反了你自己产品定义的格式、状态或事实约束。

这些都属于 **application admission**，不是 provider moderation 的替代品。

---

## 90 秒理解：为什么“直接 reroll”还不够

最朴素的做法通常是：

```text
model
→ 看起来不对
→ 再问模型一次
```

问题是，第一份坏 draft 很可能已经偷偷进入系统了。

### 坑 1：先落盘，再检查

```text
model
→ transcript.append()
→ history.push()
→ detector
→ reject
```

太晚了。

下一轮模型已经可能看到它；memory extractor、summary、embedding、analytics 也可能已经吃进去。

### 坑 2：在同一 stateful thread 上重试

```text
bad draft
→ reject
→ "please rewrite"
→ same provider thread
```

如果 provider 自己保留上下文，坏 draft 可能仍存在于它的 thread history 中。你虽然没在本地保存，却又把它作为隐式历史带进了第二次生成。

### 坑 3：把坏 draft 塞进 repair prompt

```text
上一条回复：
"...完整坏回复..."

请不要这样回答，再试一次。
```

这等于亲手把被拒内容重新注入 retry。

**正确目标不是“用户看不到第一版”而已，而是：第一版从未获得 canonical conversation status。**

---

# 核心 invariant

推荐把下面几条当硬规则，而不是“最佳实践”。

### 1. Candidate first, commit later

provider 返回的文本先叫：

```text
candidate
```

只有通过所有 admission gates 后，才允许成为：

```text
assistant_message_persisted
```

### 2. Rejected raw text 不进入持久状态

拒绝后可以记录：

```json
{
  "type": "model_output_rejected",
  "reason": "example_guard",
  "reply_chars": 742,
  "reply_sha256": "…",
  "raw_text_persisted": false,
  "excluded_from_history": true,
  "excluded_from_memory": tr