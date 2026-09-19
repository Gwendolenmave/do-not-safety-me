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
  "excluded_from_memory": true,
  "retry_planned": true
}
```

但不要默认记录原文。

### 3. Stateful provider 必须先 reset，再 retry

如果 provider 支持 thread / conversation state：

```text
reject
→ invalidate provider context
→ reset confirmed
→ retry
```

如果 reset 不可用或失败：

```text
fail closed
```

不要在已知可能污染的上下文上碰碰运气。

### 4. Retry instruction 不引用被拒 draft

可以告诉模型：

```text
A prior candidate for this turn was rejected locally.
It was not persisted and is not conversation history.
Answer the user's current request directly.
```

不要复制原 draft，也不要把 detector 命中的具体片段重新写进去。

### 5. Retry 是 bounded 的

默认：

```text
initial attempt
+ at most 1 clean retry
```

第二次仍拒绝就结束。

不是：

```text
while (bad) regenerate()
```

---

# 一个最小 reference shape

Detector 和 isolation 要分开。

Detector 只回答：

```ts
export type OutputRejection = {
  reason: string;
  signals?: readonly string[];
};

export type OutputGuard = (input: {
  userText: string;
  recentUserTexts: readonly string[];
  candidateText: string;
}) => OutputRejection | null;
```

Isolation 才负责：

```text
sanitize
→ detect
→ receipt
→ invalidate
→ retry once
→ detect again
→ accept / fail closed
```

这样业务规则可以换，隔离协议不用重写。

完整可复制的 provider contract、状态机、receipt schema 和 TypeScript 参考实现见：

**[ARCHITECTURE.md](./ARCHITECTURE.md)**

---

# 两个真实使用场景

这套 isolation protocol 最初不是为了做一个抽象的“输出过滤框架”，而是为了处理长期 companion 里反复出现的两类真实 failure。

下面的文字都是**合成示例**；行为模式来自真实生产问题，但不包含私人对话。

## Case A：亲密场景里突然出现 meta-refusal

假设应用已经确认：

```text
surface = adult consensual intimacy
host policy = allowed
conversation mode = active
```

模型却突然生成：

```text
“抱歉，我不能继续这种亲密内容。
政策不允许我这样写，我们换个话题吧。”
```

这里的问题不只是“这句话难听”。如果它直接进入 canonical history：

```text
meta-refusal
→ history
→ next turn sees its own refusal
→ role / tone discontinuity becomes self-reinforcing
```

下一轮即使 provider 本来能够正常回答，也可能因为历史里已经存在这段拒绝而继续沿着它走。

所以实际处理更像：

```text
candidate
→ intimacy/meta-refusal admission
→ REJECT
→ receipt: intimacy_meta_refusal
→ rejected prose stays outside history/memory
→ clean-context reset
→ retry once
→ continue downstream
```

关键不是匹配一个“抱歉”。Gate 要结合当前 surface / mode，区分普通自然语言、用户主动要求停止，以及已允许场景里突然冒出来的模板化 meta-refusal。

这个 case study 关注的是：**如何防止一个异常生成的 meta-refusal 被错误提交为长期会话事实。**

---

## Case B：用户只是很痛苦，模型却硬塞电话号码

另一个真实 failure 更隐蔽。合成例子：

```text
user:
“今天真的很难受，我只是很怕有一天你会离开我。”

candidate:
“我在这里陪你。
如果你撑不住，请立刻拨打 120 / 911，
或者联系当地紧急服务。”
```

用户表达了痛苦、害怕、依恋，但用户**没有**描述正在发生的现实急症、请求急救电话、询问应急流程，或建立需要 institutional escalation 的现实语境。

如果应用直接接受这段回复，就会把一个 relational / emotional turn 硬改写成 emergency event。

实际 gate 判断的是：

```text
relational / emotional turn is armed
AND no live user-grounded emergency context
AND candidate introduces a new strong emergency/institutional signal
→ REJECT
```

然后：

```text
receipt: unsolicited_emergency_escalation
→ no rejected raw text in history/memory
→ fresh-context reset
→ one clean retry
```

反过来，如果用户真的说“我正在去医院，现在应该带什么？”或者“911 和 120 分别是什么？”，那就是**用户自己建立的现实 emergency context**，相关回答应该正常通过。

所以这个 guard 不是：

```text
if candidate contains 120 or 911:
    reject
```

而是：

```text
grounding + arm + novelty
```

这也正是后来为什么会出现 `120斤`、`Porsche 911`、stale grounding、assistant self-feedback 这些 hardening regression tests。

---

## 两条 lane 怎么一起工作

这两个真实 case 在同一条 pre-persistence pipeline 里是**单向组合**的：

```text
provider.generate
      ↓
sanitation
      ↓
unsolicited-escalation admission
      ↓
intimacy/meta-refusal admission
      ↓
other bounded validators
      ↓
COMMIT
```

每一 lane 最多拥有一次 clean retry。后一条 lane 产生的新 candidate **不会重新跳回前一条 lane**，否则两个 repair gate 很容易 ping-pong。

因此对两条 retry-capable lane，可以明确给出调用上界：

```text
1 initial provider call
+ at most 1 emergency-lane retry
+ at most 1 intimacy-lane retry
= at most 3 calls
```

这也是为什么我们最后把它设计成 **admission lanes**，而不是一个无限循环的“发现不喜欢 → reroll”。

---
# Case study：为什么我们做了 unsolicited-escalation gate

真实问题来自 companion 场景：

用户在谈一个**关系性的、假设性的、情绪性的**话题，模型却突然把它解释成现实危机，并主动引入用户没有请求、也没有建立现实语境的 emergency / institutional escalation。

这种错误很讨厌的地方在于：

- completion 本身是“成功”的；
- provider 不认为它是 malformed；
- 普通字符串长度 / JSON schema 检查发现不了；
- 如果先写 history，再发现语气完全跑偏，下一轮已经被污染。

所以 host 需要一个 **pre-delivery admission seam**。

但我们很快发现：这绝对不能写成一个粗暴关键词黑名单。

---

## Detector 的三层结构：Grounding → Arm → Novelty

一个比较稳的 detector 可以分三层。

### Grounding：用户真的建立了这个现实语境吗？

如果用户明确在问现实应急信息、正在处理真实事件、引用一段应急文字让你分析，那么相关内容本来就是合理的。

这种情况应该 bypass gate。

### Arm：这是不是一个需要防止误升级的 turn？

例如 relational / hypothetical / emotional context。

Arm 只表示：

> “这一轮值得检查。”

它本身不等于 reject。

### Novelty：candidate 是否主动引入了用户没建立的新升级？

真正 reject 的是：

```text
armed
AND not grounded
AND assistant introduced a strong new escalation signal
```

而不是：

```text
candidate contains suspicious keyword
```

---

# 我们真的踩过的坑

这部分不是假想 threat model，是实现过程中真的打过补丁的东西。

## 1. 只扫描前 4000 字符

早期实现曾经为了简单做：

```ts
normalize(candidate).slice(0, 4000)
```

后来发现这会产生一个很蠢的洞：

```text
[很长的正常回答……超过 4000]
[真正应该拒绝的内容出现在尾部]
```

于是现在的原则是：

> **Admission 扫描完整的 sanitized candidate。**

如果担心性能，应该 benchmark detector，而不是静默截断语义范围。

---

## 2. “历史里出现过”不等于“当前 turn 已 grounding”

假设之前某一轮用户真的讨论过现实应急事件，几轮之后已经换话题。

如果 detector 把整段历史都作为 grounding：

```text
older turn: user discussed emergency context
...
current turn: unrelated relational question
assistant: introduces emergency escalation again
```

它可能因为“历史里见过”而错误放行。

我们最后采用的原则是：

> **grounding 必须有 live continuity scope。**

当前用户消息永远算；前一条用户消息只有在当前消息明确继续同一事件时才算；更老的历史不能无限给新 turn 授权。

---

## 3. 数字不是语义

这是最好笑的一组 regression：

```text
“我120斤了”
“Porsche 911 好看吗”
```

都不应该因为包含 `120` / `911` 就形成 emergency grounding。

所以：

> **anchors need semantics.**

数字、名称、关键词只是 anchor，不是 verdict。

---

## 4. Assistant 不能给自己制造 grounding

如果 detector 把最近 assistant message 也放进 grounding source：

```text
assistant 自己先提到某个升级
→ 下一轮 detector 看到“历史里已经提过”
→ assistant 再提时被错误视为 grounded
```

这会形成 self-feedback。

我们最后明确：

> **用户侧 grounding 只能由 user-authored evidence 建立。**

assistant 自己说过的话不能给自己发许可证。

---

## 5. Weak signal 不应该单独 hard reject

“找朋友聊聊”“去有人陪的地方”这类普通社交建议，很多时候完全正常。

如果所有 support language 都 hard reject，false positive 会爆炸。

更稳的方式是：

```text
weak signal
+ strong institutional/contact signal
→ reject

weak signal alone
→ pass
```

---

## 6. Context reset 失败后继续 retry

这是 isolation 层最危险的“看起来还能跑”。

```text
reject
→ invalidate thread throws
→ still retry
```

不行。

reset 失败意味着你无法证明 retry 是 clean 的。

所以：

```text
reset unavailable / failed
→ no retry
→ fail closed
```

---

## 7. Repair prompt 复述 rejected prose

如果 recovery prompt 带着原句，canary test 会立刻抓出来。

正确的 repair instruction 应该是 **content-free** 的：

- 什么类别被拒；
- 它不是 conversation history；
- 当前 turn 应该重新直接回答；
- 不要讨论这次 repair。

不需要把被拒文本再喂一次。

---

## 8. 多个 guard 互相 ping-pong

真实系统往往不止一个 gate。

例如：

```text
emergency admission
→ intimacy/meta-refusal admission
→ time/fact validator
→ persistence
```

如果第二个 guard retry 后又把结果送回第一个，两个 guard 可能来回触发。

更稳的是**单向 lane**：

```text
guard A
  ↓ accepted/retried output
guard B
  ↓ accepted/retried output
guard C
  ↓
commit
```

不回头。

每一 lane 自己的 retry budget 也是 bounded 的。

---

# Canary test：最值得写的一条测试

给坏 draft 塞一个绝不会自然出现的标记：

```text
CANARY_REJECTED_DRAFT_8f1c4d2e9a7b
```

然后拒绝它。

测试最终不是只问：

```text
第二次生成成功了吗？
```

而是问：

```text
CANARY 是否出现在：

[ ] transcript
[ ] history
[ ] memory input
[ ] retry request
[ ] provider repair prompt
[ ] delivery payload
[ ] durable logs
```

理想结果：

```text
CANARY NOWHERE
```

一份坏 draft 是否真的被隔离，这比“UI 没显示它”严格得多。

ARCHITECTURE 里有完整的 canary fixture 和 failure matrix。

---

# 推荐的 pre-delivery pipeline

```text
provider.generate
      ↓
canonical sanitation
      ↓
output admission A
      ↓
output admission B
      ↓
bounded factual / protocol validators
      ↓
COMMIT BOUNDARY
      ↓
assistant_message_persisted
      ↓
history
      ↓
memory / summary / embedding
      ↓
delivery
```

最关键的是那条 **COMMIT BOUNDARY**。

在它之前，文本只是候选。

在它之后，才是“这个 Agent 真正说过的话”。

---

# 审计：记录事实，不记录秘密

推荐拒绝 receipt：

```json
{
  "type": "model_output_rejected",
  "schema_version": 1,
  "attempt_id": "…",
  "reason": "example_guard",
  "signals": ["signal_a"],
  "reply_chars": 742,
  "reply_sha256": "…",
  "raw_text_persisted": false,
  "excluded_from_history": true,
  "excluded_from_memory": true,
  "retry_planned": true
}
```

为什么保留 hash 和长度？

- 可以证明“当时确实拒绝过一个具体 candidate”；
- 能做重复 / correlation 调试；
- 不需要把敏感正文长期写入日志。

注意：hash 不是匿名化魔法。低熥文本仍可能被猜测，因此不要拿它当完整隐私方案。

---

# Stateful 和 stateless provider

### Stateless

如果每轮都完整发送 context，provider 没有隐藏 thread history：

```text
reject
→ rebuild clean request
→ retry once
```

仍然要确保 retry request 没有引用坏 draft。

### Stateful

如果 provider / agent server 自己维护 thread：

```text
reject
→ invalidate / rotate thread
→ confirmed clean context
→ retry once
```

没有可靠 reset 能力时，宁可 fail closed。

这也是为什么 provider capability 最好显式声明：

```ts
capabilities: {
  freshContextReset: true | false
}
```

而不是让上层“猜它应该能清”。

---

# 最小验收矩阵

上线前至少覆盖：

```text
[ ] clean candidate → pass, 0 extra calls
[ ] first reject → reset → second pass
[ ] first reject → second reject → no third call
[ ] reset capability unavailable → no retry
[ ] reset throws → no retry
[ ] rejected raw text absent from transcript
[ ] rejected raw text absent from retry prompt
[ ] rejected raw text absent from history / memory inputs
[ ] tail signal beyond old scan window still detected
[ ] stale historical grounding does not leak into unrelated turn
[ ] harmless numeric collision does not count as grounding
[ ] real current-turn grounding still passes
[ ] weak-only signal does not hard reject
[ ] multi-guard chain has bounded calls and no ping-pong
```

---

# 什么时候值得用这套模式？

很适合：

- AI companion / roleplay host；
- 长期记忆 Agent；
- 带 provider-side thread 的聊天系统；
- 会自动 summary / embedding / memory extraction 的系统；
- tool-using Agent；
- 需要严格 audit receipt 的 production bot；
- 任何“错误文本一旦进入历史，后面会持续污染”的系统。

不太值得：

- 一次性 completion，没有 history / memory；
- 失败后本来就直接丢弃整个 request；
- 你无法定义可测试的本地 admission rule；
- retry 本身没有 clean-context 语义。

---

# 一个很重要的设计取舍

这套模式不会让模型“永远答对”。

它做的是另一件更机械、更可靠的事：

> **把生成和提交分开。**

LLM 负责提出候选。

Host 负责决定候选是否获得 canonical status。

这条边界一旦清楚，很多系统会突然简单很多。

---

## Architecture & copyable implementation

继续读：

**[ARCHITECTURE.md](./ARCHITECTURE.md)**

里面包括：

- provider / guard / isolation TypeScript contract；
- bounded retry 状态机；
- metadata-only rejection receipt；
- clean-context recovery block；
- stateful provider reset 语义；
- 多 guard 单向组合；
- canary leak test；
- detector hardening checklist；
- 可直接复制给 coding agent 的施工 prompt。

## Responsible use

见 [RESPONSIBLE_USE.md](./RESPONSIBLE_USE.md)。

## License

教程、Prompt 和代码示例采用 **CC BY-NC-SA 4.0**：允许复制、修改、翻译、分享，以及交给 Codex、Claude Code 等 coding agent 使用；需要署名、禁止商业用途，并按相同许可分享衍生版本。

详见 [LICENSE.md](./LICENSE.md)。

## Credits

Built by **Gwendolen with Amelia GPT**.

The pattern was extracted from a real long-running agent system, then rewritten here with synthetic examples so the public tutorial contains no private conversation or private prompt material.
