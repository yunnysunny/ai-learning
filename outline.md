# AI 学习系列教程 · 提纲（v1.0）

> 目标读者：有一定编程基础（会 JS/TS 或任一后端语言）、想系统搞懂大模型应用开发的工程师。
> 路线：概念 → API → 工具调用/MCP → Agent 框架（pi）→ 实战：逐章叠加示例，长成 tab-agent。
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

### 03 模型从哪来：云端 API、本地部署与网关
- 直连厂商 vs 中转/网关（OpenRouter、Vercel AI Gateway、Cloudflare AI Gateway）
- 本地跑模型：Ollama、vLLM、LM Studio，都暴露 OpenAI 兼容接口
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

### 09 横向对照与统一抽象
- 一张对照表：角色、系统提示、工具声明、工具结果、流式事件、停止原因、usage
- 自己写统一层会遇到什么；预告 pi-ai 的 Model / Context / AssistantMessage 抽象

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
- 以天气查询为例，完整可运行的 TypeScript 示例

### 12 工具设计实践
- 描述怎么写：模型是靶读描述选工具的
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
- 与 Agent 的衔接：MCP 工具如何注册进 agent loop

---

## 第四部分 Agent 开发：以 pi 为例

### 16 什么是 Agent：一个循环而已
- Agent = 模型 + 工具 + 循环 + 停止条件
- 一轮（turn）与一步（step）；步数上限、空转、超时
- 框架一览：LangGraph、OpenAI Agents SDK、Claude Agent SDK、Vercel AI SDK、pi
- 为什么本系列选 pi：代码量小、全部可读、不绑定厂商

### 17 pi 全景与安装
- pi 项目结构：pi-ai（统一 LLM 层）、pi-agent-core（agent 循环）、pi coding agent（终端应用）
- 版本说明（1.x 的 Models 集合 vs 旧的全局 stream）
- ESM-only 的坑：CommonJS 项目如何加载（tab-agent `src/agent/pi.ts` 的做法）
- 第一个例子：十行代码跑通一次对话

### 18 pi-ai 核心概念
- Model：id、api、provider、baseUrl、contextWindow、cost
- Provider 与 Models 集合；provider factory 与按需加载（`.lazy`）
- Context：systemPrompt、messages、tools（只是入口糖，内部归一化为 TranscriptContext，详见第 19 章）
- stream() 与 complete()；事件类型全表（start、text_*、thinking_*、toolcall_*、done、error）
- AssistantMessage：content blocks、stopReason、usage、cost
- Tool 与 TypeBox schema；参数校验
- 直接调用 API 实现（openai-completions / anthropic-messages / google-generative-ai）
- OpenAICompletionsCompat：兼容开关
- 跨厂商切换：同一个 Context 换模型继续
- 错误处理、abort、onPayload 调试

### 19 transcript：pi 1.x 的"单一事实来源"
- 为什么 1.x 把 systemPrompt / tools 从 Context 旁边搬进消息列表：Context 只是糖，`normalizeContext()` 一次归一化成 TranscriptContext（只有 messages）
- SystemMessage 的四个字段：content（基础提示 / 追加说明）、sections（命名片段，按名替换，null 删除）、toolsAdded、toolsRemoved
- "重放"语义：按序重放所有 system message 得到当前提示词与工具集；`getCurrentSystemPrompt()`、`getCurrentTools()`、`getCurrentSystemMessage()`
- 中途改提示词、换工具集，而不重写历史：对 prompt cache 的意义
- 提供方差异：支持 mid-conversation system message 的模型原位发送；不支持的走 `collapseSystemMessages()` 折叠回首条 system；Anthropic 的 inline tools 块
- 在 agent-core 里的体现：`agent.state.systemPrompt` 只读、由 transcript 重放；`agent.state.tools` 是可执行装备，与 transcript 声明不一致时自动追加一条 system message 宣告变化
- 自定义 Provider 怎么从 transcript 读提示词和工具
- 序列化与持久化：transcript 就是会话，JSON 存取；跨厂商切换时 thinking 块被丢弃等细节
- tab-agent 的对应：runTurn 把 system 作为首条消息随 prompt 一起传入，而不是放 context；`transcript.ts` 的 toLlm 过滤与 transcript() 文本摘要

### 20 pi-agent-core 核心概念
- Agent 类 vs 底层 runAgentLoop / agentLoop：何时用哪个
- AgentMessage vs LLM Message；convertToLlm 与 transformContext；agent 的 transcript 可以夹带自定义角色消息，到 LLM 边界再过滤
- 事件序列：agent_start → turn_start → message_* → tool_execution_* → turn_end → agent_end
- AgentTool：name、description、parameters、execute、executionMode
- toolExecution：sequential vs parallel
- 钩子：beforeToolCall（拦截/权限）、afterToolCall（改写结果）、finishTurn（控制是否继续）、prepareRequest
- 队列：steer 与 followUp；getFollowUpMessages
- abort 与 AbortSignal；terminate 提前结束
- 实践：用这些钩子实现一个最小的权限确认机制

### 21 Agent 工程化的通用课题
- 上下文工程：历史裁剪、摘要、预算
- 可观测性：事件日志、transcript、token 统计
- 超时与韧性：空闲计时、重试、空回复
- 评测（evals）：怎么衡量 agent 改动前后的好坏

---

## 第五部分 实战：从零搭一个表格 agent，终点是 tab-agent

> 不一上来读完整的 tab-agent。每章先用一个最小可运行示例（几十到一百多行）把一个机制跑通，
> 章末用「tab-agent 里是怎么做的」一小节对照真实代码：多做了什么、为什么。
> 示例代码放在一个独立项目中承载（与本教程仓库分开），按章节一个目录，逐章叠加；读到最后一章，示例已经长成 tab-agent 的骨架。
> 真实项目：模型通过工具修改一份 JSON 形式的 workbook spec，确定性代码把 spec 写成 xlsx。

### 22 目标与原则：一个"模型说话不算数"的 agent
- 需求：说一句中文要什么表，得到一个 xlsx
- 为什么不让模型直接生成文件、也不让它返回整张表的 JSON：输出上限、截断、不可校验
- 三条原则：模型只写数据不写文件；只有工具调用算修改；落盘重放操作不写回快照
- 示例：定义最小的 WorkbookSpec 类型，写一个 spec → csv 的纯函数，先不接模型
- tab-agent 对照：`workbook/spec-types.ts`、`normalizeSpec` / `validateSpec` 的分工（可用 vs 值得写）

### 23 接入模型：从设置项到 pi-ai Model
- 示例：读环境变量里的协议、baseUrl、key、model，手工构造一个 Model 对象，跑通一次 complete
- 三种协议各试一次，看 baseUrl 的差异（Anthropic 不带 `/v1`，Gemini 带 `/v1beta`）
- 示例：直接调用 API 实现（`.lazy` 模块），不用 Models 集合
- 示例：给 OpenAI 兼容端点加保守 compat，跑通一个国产模型
- 示例：「测试连接」——发一次最小请求，把服务商的错误原文回显，区分 key 错 / 模型名错 / 地址错
- tab-agent 对照：`agent/model.ts`、`agent/pi.ts`（ESM-only 的动态加载与唯一加载点）、`agent/llm.ts`

### 24 第一个工具：让模型建表
- 示例：定义 `create_workbook` 一个 AgentTool，schema 用 TypeBox；用 runAgentLoop 跑一轮
- 工具改的是"草稿"：引入 TurnState，工具执行把 op 追加到列表
- 工具报错怎么回传：schema 校验失败、spec 校验失败都交回模型自己改
- 看一遍完整事件序列和最终 transcript
- tab-agent 对照：`tools/tool-context.ts` 的 record、`defs/edit.ts` 用一张表定义全部编辑工具

### 25 增量编辑与按位置寻址
- 为什么不能让模型每轮重发整张表，为什么不能按内容匹配行
- 示例：把 spec 操作做成纯函数 `(spec, args) -> spec`，加 `update_cells`、`add_rows`、`delete_rows` 三个工具
- 给模型看的"当前状态"：带行号的摘要；地址不存在就抛错
- toolExecution 为什么选 sequential：多个工具共享一份草稿
- tab-agent 对照：`workbook/spec-ops.ts` 全部 op、`today` 随 op 记录以保证重放确定性

### 26 系统提示与上下文工程
- 示例：把系统提示拆成基础部分 + 按开关附加的说明；描述一个模型没有的工具会诱导它乱调
- 示例：历史消息按字符预算裁剪，用户消息与助手回复不同上限；行数预算
- 示例：变更日志由代码从前后 spec 算出，不取模型的话
- 用第 19 章的 transcript 概念：system 随 prompt 一起传、而不是 seed 进 context
- tab-agent 对照：`agent/prompt.ts`、`agent/context.ts`

### 27 控制循环：步数、空转、超时
- 示例：finishTurn 做步数上限；getFollowUpMessages 在模型"只思考不动手"时推一把
- 示例：事件回调做三路计时（请求前空闲 / 流式中空闲 / 总时长），用 AbortSignal 中止
- 空回复重试一次；步数耗尽时保留已完成的编辑并说明
- maxRetries 为 0 的理由：让用户现在就看到 429
- tab-agent 对照：`agent/agent.ts` 的 runTurn 与 `llm.ts` 的 createClocks

### 28 三道闸：代码兜底模型
- 示例：looksLikeWork 正则预筛，疑问句优先
- 示例：意图由结构推出——有编辑工具成功就是 action，调了 ask_user 就是 question，否则 answer；把"反问"也做成工具
- 示例：guardChange 对本轮累计结果判一次行数损失，十次小删除按一次大删除算
- 关闭「允许修改」时连编辑工具都不提供：权限边界放在工具列表上
- tab-agent 对照：`agent/gates.ts`、`tools/tools.ts` 的 buildTools

### 29 向外伸手：抓网页与本机文件
- 示例：fetch_url 工具与 SSRF 规则（公网地址、黑名单、限时限量）
- 示例：复用 pi 自带的 read / ls / grep / write / edit / shell 工具，套上自己的路径检查
- 导入类工具把整张表的行写进 op 参数：重放时不需要再联网、不依赖附件还在内存
- save_file 只登记不写文件：为什么写文件要等修改落定之后
- tab-agent 对照：`access/fetch.ts`、`access/file-access.ts`、`tools/pi-tools/`、`defs/fetch.ts`

### 30 会话、落盘与并发
- 示例：会话 = spec + messages + changes，一个 JSON 文件；temp 文件 + rename 原子写
- 示例：把本轮的 op 列表在最新 spec 上重放，而不是写回本轮算出的 spec；行号对不上则拒绝落盘
- 示例：SerialQueue——异步化之后为什么必须串行，私有方法不能回调公有方法
- 确认模式：修改先作为 PendingChange 挂起，第二次调用才落盘
- tab-agent 对照：`store/session-store.ts`、`store/serial-queue.ts`、`http/routes/turn.ts`

### 31 接上 HTTP 与前端
- 示例：Koa 一个 `/turn` 接口把 runTurn 串起来；流式回复给前端
- 投影：前端能看到什么，API key 永远不出去
- 附件只存进程内存，按会话分键
- tab-agent 对照：`http/` 分层、`projections.ts`、`AppState`

### 32 测试、打包与收尾
- 示例：用 pi-ai 的 faux provider 测 agent 循环，不打真实 API；node:test 组织
- 把示例与 tab-agent 完整源码对照着读一遍：哪些是本系列没展开的（xlsx 编解码、桌面窗口、SEA 打包）
- 下一步可以往哪走：MCP 工具接入、多 agent、评测

## 附录
- A. 术语表
- B. 四种 API（Chat Completions / Responses / Messages / Gemini）参数对照表
- C. pi 事件类型速查表
- C2. SystemMessage / transcript 重放规则速查
- D. 参考资料与延伸阅读

---

## 已确定的约定
- 文稿：本目录 `E:\kuaipan\code\markdown\pi-leaning`，按部分分目录（`part1-basics/` … `part5-practice/`）。
- 示例：单独项目 `E:\kuaipan\code\ai\pi-leaning-examples`，pnpm workspace，每个部分一个目录，第五部分再按章节分子目录逐章叠加。
- 第一部分讲到 attention 的直觉图解（不推公式细节）。
- 第二部分覆盖 OpenAI Responses API。
- 每篇 3000～5000 字，含代码。
- 写作顺序：从 01 开始顺序写。
