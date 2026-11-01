# AI 学习系列教程 · 提纲（v1.2）

> 目标读者：有一定编程基础（会 JS/TS 或任一后端语言）、想系统搞懂大模型应用开发的工程师。
> 路线：概念 → API → 工具调用/MCP → 从零写一个 agent（再对照 pi）→ 实战：逐章叠加示例，长成 tab-agent。
> 示例语言统一用 TypeScript / Node.js，与 pi 和 tab-agent 一致。

---

## 第一部分 大模型基础：先把概念和历史讲清楚

### 01 从 Transformer 到 ChatGPT：大模型简史
- 2017 Transformer 与注意力机制（只讲直觉，不推公式）
- GPT-1/2/3：预训练 + 规模定律，"涌现"是怎么被发现的
- InstructGPT / RLHF：为什么模型从"补全"变成"对话"
- 2022.11 ChatGPT 引爆；GPT-4、Claude、Gemini 三家格局形成
- 开源路线：Llama、Mistral、Qwen、DeepSeek（R1 的意义）
- 2025–2026：推理模型、长上下文、Agent 元年；国产模型现状（Qwen、Kimi、GLM、MiniMax、豆包）
- 产出：一张时间线图

### 02 使用大模型前必须懂的核心概念
- token 与分词；为什么中文和英文"价格"不同
- 上下文窗口、输入/输出 token 与计费；上下文不是记忆
- system prompt / user / assistant 角色；多轮对话是"每次重发全部历史"
- temperature、top_p、max_tokens、stop
- 流式输出（SSE）：为什么几乎所有应用都要流式
- 推理模型（thinking / reasoning）：思考内容与回答的分离
- 多模态输入（图片、文件）
- Prompt caching：缓存读写的计费差异
- 幻觉、不确定性、"模型的话不算数"——为后面的工具调用埋伏笔

### 03 模型从哪来：云端 API 与网关
- 直连厂商 vs 中转/网关（OpenRouter、Vercel AI Gateway、Cloudflare AI Gateway）
- 本地部署（Ollama、vLLM 等）硬件成本高，只一句带过
- 怎么选模型：能力、价格、延迟、上下文、是否支持工具调用
- BYOK（自带 key）类应用的设置项：协议、baseUrl、apiKey、model（tab-agent 设置页就是这四项）

---

## 第二部分 各家模型 API 的异同：同一件事三种写法

### 04 OpenAI Chat Completions：事实上的行业标准
- 请求：model、messages、temperature、stream、tools
- messages 角色：system / developer / user / assistant / tool
- 响应结构与 finish_reason；usage 字段
- 流式：SSE 分片、delta 合并、`[DONE]`
- 抓包看一次完整请求

### 05 OpenAI Responses API：新一代接口有什么不同
- input/output items、内置工具、状态保存（store）
- 与 Chat Completions 的对照；什么时候需要迁移

### 06 Anthropic Messages API
- system 是独立字段，不在 messages 里
- content blocks：text / image / tool_use / tool_result / thinking
- stop_reason 与 OpenAI finish_reason 的对应
- extended thinking 的参数与流式事件
- cache_control 显式缓存；baseUrl 不带 `/v1` 的坑

### 07 Google Gemini generateContent
- contents / parts / role（user、model）
- systemInstruction、generationConfig
- functionDeclarations 与 functionCall / functionResponse
- streamGenerateContent；baseUrl 要带 `/v1beta`

### 08 "OpenAI 兼容"的现实：国产模型与开源推理框架的差异点
- DeepSeek、Qwen、Kimi、GLM、Ollama、vLLM 各自的小坑
- developer 角色、store、strict、reasoning_content、并行工具调用支持
- 保守兼容策略：参考 tab-agent `src/agent/model.ts` 的 `CONSERVATIVE_COMPAT`
- "只返回整段 JSON、不支持 stream" 的接口怎么识别和报错

### 09 横向对照，动手写一个最小统一层
- 一张对照表：角色、系统提示、工具声明、工具结果、流式事件、停止原因、usage
- 动手：定义统一的 Message / AssistantMessage（内容块）/ 流式事件类型
- 动手：实现 openai-completions 与 anthropic-messages 两个适配器，同一段调用代码切换厂商（Gemini 留作练习）
- 这一层（示例里叫 `mini-llm`）就是第四部分自研 agent 的底座
- 预告：pi-ai 是同一思路的成熟版，第 23 章再对照它多做了什么

---

## 第三部分 工具调用与 MCP：让模型"动手"

### 10 工具调用前史：从手写 JSON 提示词到 ReAct
- 没有 function calling 时怎么做：提示词约定输出格式 + 正则解析
- ReAct 模式：Thought / Action / Observation
- 这些做法的脆弱之处

### 11 Function Calling：2023 年的分水岭
- OpenAI `functions` → `tools` 参数的演变
- 一次工具调用的完整时序：模型返回 tool_call → 调用方执行 → 回传 tool result → 模型续写
- 三家写法对比（OpenAI tools / Anthropic tool_use / Gemini functionDeclarations）
- 并行工具调用、strict 模式与 JSON Schema
- 以天气查询为例，完整可运行的 TypeScript 示例（用第 09 章的统一层，顺带给它补上工具声明与工具结果）

### 12 工具设计实践
- 描述怎么写：模型是靠读描述选工具的
- 参数 schema：枚举、必填、默认值；TypeBox / zod 生成 schema
- 错误如何回传：报错交回模型自己改，不要吞掉
- 工具结果截断、工具数量与顺序对提示词的影响
- 幂等与副作用：读工具与写工具分开

### 13 MCP：把工具从应用里拆出去
- 2024.11 Anthropic 发布 MCP 的动机：复用、跨语言、低改造成本
- 架构：Host / Client / Server；JSON-RPC 2.0
- 三种传输：stdio、HTTP+SSE、Streamable HTTP，各自适用场景
- 原语：tools、resources、prompts、sampling（哪些被广泛支持）
- 协议演进时间线

### 14 动手：写一个 MCP Server 和 Client
- 用官方 SDK 写天气 server，两种传输各一版
- MCP Inspector 调试
- Client：连接、列出工具、转换为 OpenAI tools 格式、循环处理工具调用
- 抓包看 JSON-RPC 请求与 SSE 响应

### 15 MCP 的安全与取舍
- 鉴权：示例服务没有鉴权，公开部署要加中间件 / OAuth
- 工具描述投毒：第三方 MCP 的描述会进入提示词
- 什么适合做成 MCP、什么适合本地 tools
- 与 Agent 的衔接：MCP 工具如何注册进 agent loop（第四部分的 mini-agent 会接入第 14 章的 server）

---

## 第四部分 Agent 开发：从零写一个 agent，再对照 pi

> 不一上来就学 pi。pi 是成熟项目，源码里大量逻辑服务于健壮性和多厂商兼容，拿来讲原理会绕。
> 本部分在第 09 章统一层之上自己写一个 `mini-agent`，每章只加一个机制；章末「pi 怎么做」一小节对照 pi 的对应设计。
> 第 23 章专讲 pi 的用法：用 pi 把 mini-agent 重写一遍，看成熟框架多做了什么。

### 16 什么是 Agent：一个循环而已
- Agent = 模型 + 工具 + 循环 + 停止条件；一轮（turn）与一步（step）
- 框架一览：LangGraph、OpenAI Agents SDK、Claude Agent SDK、Vercel AI SDK、pi
- 为什么先自己写：先把机制写透，再看框架就知道它多做了什么、为什么
- 本部分路线图：mini-agent 逐章长出来的样子

### 17 最小 agent 循环
- 示例：几十行的循环——发请求 → 有工具调用就执行并回传 → 没有就结束
- 停止条件：stopReason、步数上限；空转与超时先列为已知问题
- 和第 11 章手写工具调用的区别：把"循环"从业务代码里抽出来
- 不足清单：没有事件、工具没校验、外部插不了手、提示词和工具不能中途变（后面几章逐个解决）

### 18 消息与事件：agent 的数据结构
- AgentMessage vs LLM Message：agent 内部可以夹带自定义消息（通知、UI 状态），到 LLM 边界再 convertToLlm 过滤
- 事件流：agent_start → turn_start → message_start / update / end → tool_execution_start / end → turn_end → agent_end
- 为什么用事件而不是返回值：流式 UI、日志、测试都订阅同一条流
- 示例：把统一层的 delta 事件转成 message_update；写一个控制台渲染器
- pi 怎么做：pi-agent-core 的事件序列与这里几乎一一对应

### 19 工具系统：定义、校验、执行
- AgentTool 接口：name、description、parameters（TypeBox）、execute
- 参数校验：失败不抛给调用方，而是作为工具结果交回模型
- 执行模式：sequential vs parallel；工具共享状态时为什么要串行
- 工具结果：给模型的文本 + 给 UI 的结构化 details
- 示例：read_file / write_file / calc 三个工具；接入第 14 章的 MCP server
- pi 怎么做：AgentTool、executionMode、pi 自带的 coding 工具

### 20 钩子与控制：让外部插手循环
- beforeToolCall（拦截 / 权限）、afterToolCall（改写结果）、finishTurn（是否继续）、prepareRequest（改请求）
- abort：AbortSignal 一路传到 fetch；terminate 提前结束
- steer 与 followUp：运行中插话 vs 结束后追加；getFollowUpMessages
- 示例：最小的权限确认机制（写文件前在终端问 y/n）
- pi 怎么做：pi-agent-core 的同名钩子与队列

### 21 transcript：把会话做成单一事实来源
- 问题：系统提示、工具集中途会变；改写开头会破坏 prompt cache，历史也无从追溯
- 设计：system message 作为追加记录进入消息列表——content（基础提示 / 追加说明）、sections（命名片段，按名替换，null 删除）、toolsAdded、toolsRemoved
- 重放：按序重放所有 system message 得到当前提示词与工具集；可执行的工具与声明不一致时自动追加一条 system message
- 提供方差异：支持中途 system message 的模型原位发送，不支持的发送前折叠回首条 system
- 序列化：transcript 就是会话，JSON 存取；跨厂商切换时 thinking 块的处理
- 示例：给 mini-agent 加 transcript 与重放，中途切换工具集，观察缓存命中
- pi 怎么做：pi 1.x 的 SystemMessage、`normalizeContext()` 与 TranscriptContext、`getCurrentSystemPrompt()` / `getCurrentTools()`、`collapseSystemMessages()`、Anthropic 的 inline tools 块；`agent.state.systemPrompt` 为什么只读

### 22 记忆：让 agent 记住会话之外的东西
- 分清三层：上下文（本次请求里的内容）、会话（transcript 的持久化）、长期记忆（跨会话的事实、偏好与经验）
- 短期侧：上下文快满时的压缩（compaction）——滑动窗口、滚动摘要；摘要也是"模型的话"，可能失真
- 长期记忆的三种做法：文件式（启动时注入 `AGENTS.md` / `CLAUDE.md` 这类说明文件）、工具式（给模型 `remember` / `recall` 工具，由它决定存什么）、检索式（向量库 + RAG，按相关性召回）
- 写入策略：谁决定存什么、过期与纠错；记忆投毒（与第 15 章的工具描述投毒同类）
- 记忆放进提示词的位置：用第 21 章的 transcript 追加，而不是改写开头，以免破坏 prompt cache
- 示例：给 mini-agent 加基于 JSON 文件的 `remember` / `recall` 工具，以及超出预算自动摘要的 compaction
- pi 怎么做：transformContext 做压缩、pi coding agent 加载 AGENTS.md 项目说明（写作时以 pi 源码核实）

### 23 换成 pi：成熟框架多做了什么
- pi 项目结构：pi-ai（统一 LLM 层）、pi-agent-core（agent 循环）、pi coding agent（终端应用）；1.x 的 Models 集合 vs 旧的全局 stream
- ESM-only 的坑：CommonJS 项目如何加载（tab-agent `src/agent/pi.ts` 的做法）
- 用 pi 把第 17～22 章的 mini-agent 重写一遍，附概念映射表（mini-llm / mini-agent → pi-ai / pi-agent-core）
- pi-ai 多出来的：Model 元数据（contextWindow、cost）、Provider 与 `.lazy` 按需加载、OpenAICompletionsCompat、usage 与成本、onPayload 调试、faux provider
- pi-agent-core 多出来的：Agent 类 vs runAgentLoop、transformContext、更完整的错误与中止处理
- 什么时候自己写、什么时候用框架

### 24 Agent 工程化的通用课题
- 上下文工程：历史裁剪、预算（压缩与长期记忆见第 22 章）
- 可观测性：事件日志、transcript、token 统计
- 超时与韧性：空闲计时、重试、空回复
- 评测（evals）：怎么衡量 agent 改动前后的好坏

---

## 第五部分 实战：从零搭一个表格 agent，终点是 tab-agent

> 不一上来读完整的 tab-agent。每章先用一个最小可运行示例（几十到一百多行）把一个机制跑通，
> 章末用「tab-agent 里是怎么做的」一小节对照真实代码：多做了什么、为什么。
> 示例以第四部分的 mini-agent 为底座，放在独立项目中按章节一个目录、逐章叠加；读到最后一章，示例已经长成 tab-agent 的骨架。
> tab-agent 本身基于 pi，对照小节会同时指出对应的 pi API。
> 真实项目：模型通过工具修改一份 JSON 形式的 workbook spec，确定性代码把 spec 写成 xlsx。

### 25 目标与原则：一个"模型说话不算数"的 agent
- 需求：说一句中文要什么表，得到一个 xlsx
- 为什么不让模型直接生成文件、也不让它返回整张表的 JSON：输出上限、截断、不可校验
- 三条原则：模型只写数据不写文件；只有工具调用算修改；落盘重放操作不写回快照
- 示例：定义最小的 WorkbookSpec 类型，写一个 spec → csv 的纯函数，先不接模型
- tab-agent 对照：`workbook/spec-types.ts`、`normalizeSpec` / `validateSpec` 的分工（可用 vs 值得写）

### 26 接入模型：从设置项到 Model
- 示例：读环境变量里的协议、baseUrl、key、model，构造统一层的 Model，跑通一次 complete
- 三种协议各试一次，看 baseUrl 的差异（Anthropic 不带 `/v1`，Gemini 带 `/v1beta`）
- 示例：给 OpenAI 兼容端点加保守兼容开关，跑通一个国产模型
- 示例：「测试连接」——发一次最小请求，把服务商的错误原文回显，区分 key 错 / 模型名错 / 地址错
- tab-agent 对照：`agent/model.ts`（pi-ai Model 与 compat）、`agent/pi.ts`（ESM-only 的动态加载与唯一加载点、`.lazy` 模块）、`agent/llm.ts`

### 27 第一个工具：让模型建表
- 示例：定义 `create_workbook` 一个 AgentTool，schema 用 TypeBox；用 mini-agent 跑一轮
- 工具改的是"草稿"：引入 TurnState，工具执行把 op 追加到列表
- 工具报错怎么回传：schema 校验失败、spec 校验失败都交回模型自己改
- 看一遍完整事件序列和最终 transcript
- tab-agent 对照：`tools/tool-context.ts` 的 record、`defs/edit.ts` 用一张表定义全部编辑工具

### 28 增量编辑与按位置寻址
- 为什么不能让模型每轮重发整张表，为什么不能按内容匹配行
- 示例：把 spec 操作做成纯函数 `(spec, args) -> spec`，加 `update_cells`、`add_rows`、`delete_rows` 三个工具
- 给模型看的"当前状态"：带行号的摘要；地址不存在就抛错
- toolExecution 为什么选 sequential：多个工具共享一份草稿
- tab-agent 对照：`workbook/spec-ops.ts` 全部 op、`today` 随 op 记录以保证重放确定性

### 29 系统提示与上下文工程
- 示例：把系统提示拆成基础部分 + 按开关附加的说明；描述一个模型没有的工具会诱导它乱调
- 示例：历史消息按字符预算裁剪，用户消息与助手回复不同上限；行数预算
- 示例：变更日志由代码从前后 spec 算出，不取模型的话
- 用第 21 章的 transcript：system 随 prompt 一起传、而不是 seed 进 context
- tab-agent 对照：`agent/prompt.ts`、`agent/context.ts`、`transcript.ts` 的 toLlm 过滤与 transcript() 文本摘要

### 30 控制循环：步数、空转、超时
- 示例：finishTurn 做步数上限；getFollowUpMessages 在模型"只思考不动手"时推一把
- 示例：事件回调做三路计时（请求前空闲 / 流式中空闲 / 总时长），用 AbortSignal 中止
- 空回复重试一次；步数耗尽时保留已完成的编辑并说明
- maxRetries 为 0 的理由：让用户现在就看到 429
- tab-agent 对照：`agent/agent.ts` 的 runTurn 与 `llm.ts` 的 createClocks

### 31 三道闸：代码兜底模型
- 示例：looksLikeWork 正则预筛，疑问句优先
- 示例：意图由结构推出——有编辑工具成功就是 action，调了 ask_user 就是 question，否则 answer；把"反问"也做成工具
- 示例：guardChange 对本轮累计结果判一次行数损失，十次小删除按一次大删除算
- 关闭「允许修改」时连编辑工具都不提供：权限边界放在工具列表上
- tab-agent 对照：`agent/gates.ts`、`tools/tools.ts` 的 buildTools

### 32 向外伸手：抓网页与本机文件
- 示例：fetch_url 工具与 SSRF 规则（公网地址、黑名单、限时限量）
- 示例：read / ls / grep 等文件工具，套上自己的路径检查（tab-agent 直接复用 pi 自带的工具）
- 导入类工具把整张表的行写进 op 参数：重放时不需要再联网、不依赖附件还在内存
- save_file 只登记不写文件：为什么写文件要等修改落定之后
- tab-agent 对照：`access/fetch.ts`、`access/file-access.ts`、`tools/pi-tools/`、`defs/fetch.ts`

### 33 会话、落盘与并发
- 示例：会话 = spec + messages + changes，一个 JSON 文件；temp 文件 + rename 原子写
- 示例：把本轮的 op 列表在最新 spec 上重放，而不是写回本轮算出的 spec；行号对不上则拒绝落盘
- 示例：SerialQueue——异步化之后为什么必须串行，私有方法不能回调公有方法
- 确认模式：修改先作为 PendingChange 挂起，第二次调用才落盘
- tab-agent 对照：`store/session-store.ts`、`store/serial-queue.ts`、`http/routes/turn.ts`

### 34 接上 HTTP 与前端
- 示例：Koa 一个 `/turn` 接口把 runTurn 串起来；流式回复给前端
- 投影：前端能看到什么，API key 永远不出去
- 附件只存进程内存，按会话分键
- tab-agent 对照：`http/` 分层、`projections.ts`、`AppState`

### 35 测试、打包与收尾
- 示例：写一个假的模型适配器测 agent 循环，不打真实 API；node:test 组织（对照 pi-ai 的 faux provider）
- 可选：把示例底座从 mini-agent 换成 pi，与 tab-agent 对齐
- 把示例与 tab-agent 完整源码对照着读一遍：哪些是本系列没展开的（xlsx 编解码、桌面窗口、SEA 打包）
- 下一步可以往哪走：MCP 工具接入、多 agent、评测

## 附录
- A. 术语表
- B. 四种 API（Chat Completions / Responses / Messages / Gemini）参数对照表
- C. mini-agent 与 pi 的事件 / API 对照表
- C2. SystemMessage / transcript 重放规则速查
- D. 参考资料与延伸阅读

---

## 已确定的约定
- 文稿：本目录 `E:\kuaipan\code\markdown\pi-leaning`，按部分分目录（`part1-basics/` … `part5-practice/`）。
- 示例：单独项目 `E:\kuaipan\code\ai\ai-learning-examples`，pnpm workspace，每个部分一个目录，第五部分再按章节分子目录逐章叠加。
- 第四部分先自己写 mini-agent（底座是第 09 章的统一层 mini-llm），每章末对照 pi；pi 的用法集中在第 23 章。第五部分以 mini-agent 为底座。
- 第一部分讲到 attention 的直觉图解（不推公式细节）。
- 第二部分覆盖 OpenAI Responses API。
- 每篇 3000～5000 字，含代码。
- 写作顺序：从 01 开始顺序写。
