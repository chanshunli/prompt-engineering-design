# 把提示词当代码写：Prompt Engineering 的第一性原理

> "让提示词更像编程语言，减少降维变换的损失，甚至直接用 Python 语言本身作为提示词。"
> —— 本项目 README 第一性原理之一

> 这篇文章只讲一层：**提示词本身怎么写**。
> 至于提示词之外的那套循环、工具、记忆——也就是 **harness / ReAct**——我在另外两篇里写透了，本文不再重复，只在该衔接的地方给链接。

---

## 〇、一个反直觉的开场

大多数人对 Prompt Engineering 的理解停留在"把话说清楚"。于是出现了一堆玄学：加个"请你扮演专家"、结尾来一句"这很重要"、再不行就"深呼吸一步步想"。

这些技巧有用，但它们都没回答一个更根本的问题：

> **为什么"把话说清楚"这件事本身这么难？**

答案是：**自然语言是有损的**。你脑子里那个精确的需求是高维的、带结构的；你用一句中文把它说出来时，结构被压扁了；模型再把这句中文"翻译"回它要执行的高维结构时，又得猜一次。**一来一回两次降维，损失叠加，结果就跑偏。**

理解了这一点，README 里那十几条看似零散的"第一性原理"，其实是同一条主线的不同侧面。这篇文章就把这条主线拎出来：

> **提示词工程的本质，是控制"人的意图 → 模型的执行"这条翻译链路上的降维损失。**
> **而控制损失最有效的手段，是让提示词越来越像代码。**

---

## 一、第一性原理：Transformer 最强的是翻译

先把地基打牢。

Transformer 这个架构，它**最擅长、最稳定、最不容易出错的能力是翻译**——准确说，是**高维向量到高维向量的、保结构的映射**。机器翻译是它的起家本行，但"翻译"远不止语言之间：

- 把英文翻成中文，是翻译；
- 把自然语言描述翻成 Python 代码，是翻译；
- 把 Python 代码翻成 Rust，是翻译；
- 把一段需求翻成一个 JSON 结构，也是翻译。

**只要源和目标都是"结构清晰、信息密度高"的表示，Transformer 的翻译就既稳又准。** 反过来，源头越模糊（一句没头没尾的口语），它要补的结构越多，幻觉就越多。

这给了我们第一条可操作的推论：

> **不要让模型从"模糊"翻译到"精确"，那是它最吃力的方向。**
> **要让模型从"精确"翻译到"精确"——也就是把你的提示词，提前写成接近代码的精确结构。**

这就是 README 那句"让提示词更像编程语言"的全部理由。它不是审美偏好，是顺着 Transformer 的能力梯度在走。

---

## 二、Prompt as Code：把提示词写成代码

既然要"精确到精确",最极端的做法就是**直接用代码当提示词**。本项目 README 给了好几种梯度，从弱到强排一下。

### 2.1 最强形态：直接用 Python 代码 + 填空

让模型"补全代码"，而不是"回答问题"。代码的语法本身就是一副最严格的约束模具：

```python
## Your problem {you question}, you MUST fill code in the blank:
def get_weather():
    api_weather({write you location xyz})
```

或者更进一步，把"可调用的函数"提供给模型，逼它只能在给定函数里组合答案：

```python
# Please complete the code according to the context, keep it concise.
# Note:
# 1. Only output the code content, no explanations.
# 2. Only functions provided by the context are allowed.
# 3. Only one line of code is allowed.

def get_weather(location: str, time: str) -> List[str]:
    """Query the weather for a specific location and time."""

def answer():
    """ {{QUESTIONS}}, should call function """
```

注意这里发生了什么：**类型签名 `(location: str, time: str) -> List[str]` 就是提示词的一部分**。你没写一句"请注意参数类型"，但模型已经被签名约束住了。这是把"自然语言里要啰嗦半天的约束"，压进了"代码里一行就说清的结构"。这就是降维损失最小的写法。

> ⚠️ 代价：代码输出需要 `ast` 解析、或正则抽取代码块，偶尔不稳定。README 也老实承认了"json output parse sometimes unstable"。所以——见下一条。

### 2.2 结构化输出：用括号 / JSON / 代码块锁住边界

模型的输出要能被下游程序消费，就必须有**机器可解析的边界**。README 的经验是：

> 只有用括号、JSON、或 Python 代码块（```json）包起来的输出，才能稳定地被抽取出来。

所以约定一个铁律式的输出尾巴：

```txt
MUST be **LAST output json** like below format:
```json
{"file": ..., "name": ...}
```
```

配一个朴素但够用的抽取器：

```python
def parse_code_blocks(markdown):
    code_blocks = re.findall(r'```json(?:[a-zA-Z]*)\n([\s\S]*?)\n```', markdown)
    if not code_blocks:
        raise ValueError("No code blocks found")
    return [b.strip() for b in code_blocks]
```

**输出格式 = 函数的返回类型。** 你不会允许一个函数"有时返回 dict，有时返回一段散文"，提示词的输出也不该。

### 2.3 绝对语气 = 类型约束

README 里反复强调"拒绝模糊语气，用绝对语气如 MUST"，还专门有一节叫 *Grammar capitalization highlights tone*：

```txt
Step1. you first MUST find xxx
*MUST NOT* include xxx
Every *BEGIN/END block* must use this format:
```

为什么大写、为什么 `MUST` / `MUST NOT`？因为在"翻译"的视角下，**"尽量""最好""如果可以"这类词是噪声**——它们在目标侧没有对应的确定结构，模型只能概率性地猜要不要执行。而 `MUST` 像一个布尔断言，把概率压到接近 1。

**模糊词之于提示词，等于隐式类型转换之于代码：能省则省，越显式越安全。**

### 2.4 多变量约定：`{func}` 与 `(var)`

README 提了一个很妙的小约定，呼应 Markdown 的设计哲学（`[text](link)`、`[[d-link]]`、`((block)`）：

```text
{funcabc}  →  花括号是"函数 / 占位结构"
(xyz)      →  圆括号是"变量 / 要填的值"
**{INPUT}** →  加粗 + 包裹，强调这是关键输入
```

```python
"""
Step1: Your problem is **{INPUT}**, ...
Step2: Answer this question **{INPUT}**, ...
"""
```

这一步看着小，意义却大：**你给提示词引入了"变量"和"作用域"**。一旦有了变量，提示词就不再是一段死文本，而是一个可以被参数化、被复用、被组合的**模板函数**。这直接通向下一节。

---

## 三、Lambda 演算：提示词是可组合的函数

README 有一条很硬核的原则：

> "Lambda calculus uses the same idea to calculate prompt words."

意思是：**把每一段提示词当成一个纯函数**，输入是变量，输出是文本；然后用组合、套用、复用去搭出复杂行为，而不是写一坨巨型 prompt。

README 里那些命名好的消息常量就是这个思路的雏形：

```python
content_gpt_edits      = 'I committed the changes with git hash {hash} & msg: {message}'
content_gpt_no_edits   = "I didn't see any properly formatted edits in your reply?!"
lazy_prompt = """You are diligent and tireless!
You NEVER leave comments describing code without implementing it!
You always COMPLETELY IMPLEMENT the needed code!
"""
```

每一段都是一个"小提示词函数"，带着自己的占位变量、负责一件极小的事。需要时拼装，不需要时放着。**提示词工程到这一步，已经从"写作"变成了"软件设计"。**

---

## 四、一个 system role 只做一件事：粒度即稳定性

接着上一节最关键的工程纪律，也是 README 我最认同的一条：

> "A system role prompt does one thing well. The smaller the granularity, the more stable it is."

为什么粒度越小越稳？还是翻译视角：**一个职责单一的 prompt，它的"目标结构"是收敛的、可预测的**；一个又要查库、又要写代码、又要润色文案的巨型 prompt，目标结构发散，模型每一步都在多个意图之间摇摆，损失累积得飞快。

看 README 里 `shell_gpt` 的角色定义——干净得像一个 Unix 小工具：

```python
SHELL_ROLE = """
You are Shell Command Generator
Provide only xonsh commands for {os} without any description.
If there is a lack of details, provide most logical solution.
Ensure the output is a valid shell command.
If multiple steps required try to combine them together using &&.
Provide only plain text without Markdown formatting.
Do not provide markdown formatting such as ```.
"""
```

它只做一件事：**自然语言 → 一条合法 shell 命令**。没有寒暄、没有解释、没有 Markdown。输入输出都被压到最窄。这种"小而专"的 role，正是可以被批量组合、可以被便宜模型驱动、可以被无人值守地串进流水线的前提。

> 这里和 harness 那篇文章里的 **subagent（每个子智能体只拿一小撮精准工具）** 是同构的——把"专一职责 + 精准接口"这条原则,从工具层递归地用到了提示词层。详见：[Harness 即一切](./harness-is-everything.md)。

---

## 五、多提示词自动协作：拒绝手工搬运

README 里有一句话，是整本"提示词设计"的灵魂：

> "The most important thing about Prompt Engineering is to realize **automatic collaboration** of input and output of multiple prompt words, don't output and sort it out manually."

翻译过来：**最重要的不是把单个提示词调到完美，而是让多个提示词的输入输出自动地接起来——你不要做那个手工复制粘贴、手工整理中间结果的人。**

这正是第二、三、四节那一切结构化努力的**目的所在**：

- 输出锁成 JSON（§2.2）→ 是为了让下一个 prompt 能直接吃；
- 提示词参数化成函数（§3）→ 是为了能被程序自动填值调用；
- role 拆到足够小（§4）→ 是为了能像零件一样被流水线串起来。

README 还专门提到中间结果的传递纪律：

```python
# SUMMARY & SORTING：把上下文压缩后传给下一个 agent 或用户
finally end MUST SUMMARY of what the code does
```

每一棒交接前先 **SUMMARY**，把噪声滤掉、只把精华传下去——这既省 token，又防止上下文污染。

> 这条"自动协作"的原则，往前再走一步就是 **ReAct 的 Thought→Action→Observation 闭环**，以及 harness 怎么把这套循环工业化。那是另一层的事，完整论述见：[从 CoT 到 ReAct，再到"会自己思考"的模型](./In-depth-analysis-of-ReAct.md)。本文到此为止，只强调一点：**结构化的提示词，是自动协作的前置条件。** 输出是散文，就没法自动接棒。

---

## 六、检索足够的外部上下文：UPDATE CONTEXT 模式

README 另一条第一性原理：

> "One of the greatest significances of Prompt Engineering is to **retrieve enough effective and accurate external context.**"

模型权重里没有你公司今天的数据、没有那个内部文档、没有这次报错的现场。**再精确的提示词，喂的是空气，也只能产出空气。** 所以"把对的上下文检索进来"和"把提示词写精确"是同等重要的两件事。

README 给了一个很优雅的"自知之明"模式——让模型在上下文不足时**主动喊话**，而不是硬编：

```python
"""You're a retrieve augmented chatbot. You answer based on your own
knowledge and the context provided by the user.
If you can't answer with or without the current context, you should
reply exactly `UPDATE CONTEXT`.
You must give as short an answer as possible.

User's question is: {input_question}
Context is: {input_context}
"""
```

`UPDATE CONTEXT` 这个固定 token 就是一个**控制信号**：外层程序一旦收到它，就去检索更多资料、补进 `{input_context}`、重新调用。**提示词在这里不只是产出答案，还在驱动一个检索循环。** 这又一次回到 §2.2 —— 用一个机器可识别的固定输出，把"模型"和"程序"焊在一起。

---

## 七、把"一次问对"换成"多次逼近"

人类工程师调试代码不指望一遍跑通，提示词也别指望一次问对。README 有三条原则讲的是同一件事：

> - "Multiple executions, retries on failure, and loops for a goal."
> - 复杂问题先用简单问题测，用返回结果纠正提示词再问——相当于 **"a belief network or GPT fingerprint"**。
> - 复杂算法先生成 Python 代码，多次确认正确后，再用 GPT 翻译成其它语言。

### 7.1 GPT Fingerprint：用简单问题校准提示词

这条特别值得展开。面对一个很复杂的问题，**不要一上来就把全部复杂度砸给模型**。先用一个你**已知答案**的简单问题去问，看模型怎么回、它的"思路指纹"是什么样：

- 如果简单问题它都答歪 → 说明你的提示词框架本身有问题，改框架；
- 如果简单问题答对了 → 用它的回答风格、它理解的措辞，**反过来修订你的提示词**，再上复杂问题。

这就是 README 说的 **belief network / GPT fingerprint**：你在用一组小探针，去标定模型对你这套提示词的"响应曲线"，然后顺着曲线把真正的问题喂进去。本质是**把提示词调优变成一个有反馈的迭代过程**，而不是一锤子买卖。

### 7.2 先写 Python，再翻译成目标语言

对复杂的算法/逻辑题，README 的路径是：

> **先让模型生成 Python**（生态最熟、模型最擅长、最容易验证），跑几遍确认逻辑正确，**再让模型把这段已验证的 Python 翻译成 Rust / Go / Java。**

这又回到了第一节的主线：**翻译是 Transformer 最稳的能力。** 与其让它直接生成不熟的 Rust（一步到位、难验证），不如把任务拆成"生成 Python（可验证）+ 翻译（最稳）"两段。**把不确定性集中到一个可验证的中间表示上**——Python 就是那个中间表示。

---

## 八、MRE：最小可复现样例，是最强的"例子引导"

README 压轴的一条：

> "The most effective way to solve complex code issues is to find a **minimal reproducible example** and let the AI generate what you need based on that pattern."

为什么 MRE 这么强？因为**一个好例子的信息密度，远高于一段描述**。你描述"我想要一个带重试和指数退避的 HTTP 客户端"，模型要补一堆结构；你直接给它一个 20 行的最小可跑样例，让它"照这个 pattern 改/扩"——它要做的又变回了**翻译**（从一个已知结构映射到一个相邻结构），而不是凭空创造。

MRE 还有一个隐藏好处：**它顺手把上下文也最小化了**。没有无关代码的干扰，模型不会被噪声带偏。这与 §4"粒度越小越稳"、§6"精准上下文"是同一个道理在不同尺度上的回声。

> 把"给一个最小例子让模型照着干"这件事放大到整个研究流程，就是 Karpathy 的 `autoresearch`、就是"训练模型本身也是一种逆向工程"——那条线我在 harness 那篇里展开了，这里不重复。

---

## 九、边界：提示词工程的尽头是微调

最后要诚实地划一条线。README 写道：

> "The limit of Prompt Engineering is to **fine tune the model by yourself**, and use the training data to QA the DL descriptors that are closer."

提示词工程是有上限的。当你发现：

- 同一类任务你要反复贴几千 token 的 few-shot；
- 某种领域措辞模型怎么都对不准；
- 你想要的输出分布，本质上不在预训练分布里——

那就说明你已经顶到了**"in-context（上下文内）"这条路的天花板**。再往上，就该把这些经验**从提示词里搬进权重里**——也就是微调 / SFT，用训练数据让模型的内部"描述符"更贴近你的领域。

这条边界，恰好和 ReAct 那篇文章的主线接上了：**"怎么想"正越来越多地从 prompt 下沉进模型权重**（CoT → o1 → R1）。提示词工程不会消失，但它和"训练"之间的分界线一直在移动。今天要写一长串提示词才能逼出的行为，明天可能一个微调就内化了。**好的提示词工程师，要清楚自己什么时候该停下来去标数据。**

---

## 十、收束：提示词工程像 Web 设计

README 里有一句很轻巧但很准的比喻：

> "Prompt Engineering is like web design, rich in various elements and combinations."

把全文压成一张表，你会发现这十几条原则，全是"用代码的纪律去组织文本"的不同元素：

| README 原则 | 翻译视角下的本质 | 类比的编程概念 |
|---|---|---|
| 提示词更像编程语言 | 减少"模糊→精确"的降维损失 | 强类型 |
| 绝对语气 MUST | 把概率压到 1 | 断言 / 类型约束 |
| 直接用 Python 当 prompt | 让任务退化成"翻译/填空" | 代码模板 |
| JSON / 代码块输出 | 给输出一个可解析边界 | 返回类型 |
| `{func}` `(var)` 变量 | 提示词参数化 | 函数签名 |
| Lambda 计算提示词 | 小提示词的组合 | 纯函数组合 |
| 一个 role 做一件事 | 目标结构收敛 → 稳 | 单一职责 |
| 多提示词自动协作 | 输出自动接棒，不手工搬 | 管道 / 函数复合 |
| 检索精准上下文 | 给翻译喂对源数据 | 依赖注入 |
| 多次执行 / 重试 / 循环 | 把一次问对换成迭代逼近 | 测试驱动 |
| GPT fingerprint | 用简单探针标定响应曲线 | 单元测试 / 二分定位 |
| 先 Python 再翻译 | 把不确定性收到可验证中间表示 | 中间表示 / 编译 |
| 最小可复现样例 | 用高密度例子代替低密度描述 | 样例驱动 |
| 尽头是微调 | in-context 顶到天花板就进权重 | 从配置到编译 |

一句话总结整篇：

> **Transformer 最稳的能力是翻译。所以最好的提示词，是把你脑子里那个高维、精确、带结构的意图，提前写成接近代码的样子——让模型只需要"翻译"，而不需要"猜"。**

剩下那一层——循环、工具、记忆、子智能体——是 harness 的事。两篇姊妹文在这里：

- [Harness 即一切：模型不是不够聪明，而是是否有足够有用的 tools](./harness-is-everything.md)
- [从 CoT 到 ReAct，再到"会自己思考"的模型](./In-depth-analysis-of-ReAct.md)

提示词把单点写精确，harness 把多点串起来。**两者合起来，才是完整的"用自然语言编程"。**

---

*本文所有提示词片段均摘自本仓库 [`README.md`](../README.md)。*
