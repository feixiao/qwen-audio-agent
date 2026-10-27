# Qwen-Audio-Agent 技术方案深度解析与实战指南：聚焦 AI 助手与智能座舱

> **项目定位**：[Qwen-Audio-Agent](https://github.com/QwenAudio/qwen-audio-agent) 是由阿里通义语音团队（Alibaba Token Foundry / QwenAudio Team）开源的**实时全双工语音运行时框架（Realtime Voice Runtime）**。其核心设计目标是解决传统“语音 + LLM/Agent”系统中“一调工具就沉默、一跑任务就卡死”的痛点，通过**前台对话 + 编排调度 + 后台执行**的三层解耦架构，让 AI 助手实现**“持续说话、持续工作、持续在场（Agent Presence）”**。

---

## 一、 核心痛点与架构总览：为什么 AI 助手与智能座舱需要这套方案？

在传统的语音助手（无论是桌面 AI 助手还是车载语音系统）中，对话流与任务执行流通常是**串行耦合**的：
1. **调用阻塞**：当用户要求“帮我查一下附近评分最高且有充电桩的粤菜馆并规划路线”或“重构这段代码并跑通测试”时，模型开始调用工具或多步推理，语音通道被迫挂起，用户只能面对漫长的静音等待。
2. **打断即丢失**：在全双工交互中，如果用户中途开口插话或切换话题，传统系统往往会连同正在执行的后台任务一起中断取消，导致已完成的工作白费。
3. **车控与界面状态割裂（座舱痛点）**：在智能座舱中，驾驶员可能一边通过语音下达指令，一边顺手在中控屏上修改了导航目的地或调低了空调温度。传统语音系统缺乏统一的环境事件感知，极易基于过时的上下文做出错误响应。

为彻底解决上述问题，[Qwen-Audio-Agent 技术报告（arXiv:2609.25195）](https://arxiv.org/abs/2609.25195) 提出了**前后台分离的三层核心架构（Foreground–Background Architecture）**，将“实时交互”、“确定性状态机调度”与“复杂长耗时任务执行”拆解在不同的时间尺度上协同运作。

### 1.1 逻辑三层架构与宿主组件拆解

在用户感知层面，始终面对的是**同一个语音助手**；但在系统内部，职责被严格划分为逻辑三层与两大外围支撑组件（详见 [架构解析文档](https://www.nxrte.com/jishu/71113.html) 与 [三层架构源码剖析](https://blog.csdn.net/gitblog_00133/article/details/166903254)）：

| 架构层级 / 组件 | 核心角色定位 | 核心职责与能力边界 | 关键源码目录映射 |
| :--- | :--- | :--- | :--- |
| **第一层：Frontend Agent（实时前台）** | **负责“聊”**（全双工对话与轻量操作） | 处理流式音视频输入、自然打断（Barge-in）、维持低延迟多轮对话；对可秒答的问题直接回应，对轻量或高实时指令（如车控、时间查询、记忆读写）直接调本地工具；对耗时多步任务通过 `spawn_thinking` 委派给后台。**严禁直接操作后台 Session 或子 Agent。** | `server/src/frontend/`、`server/src/voice/`、`config/frontend-agent/PROMPT.md` |
| **第二层：Orchestration Runtime（编排运行时）** | **负责“管”**（纯代码确定性调度中枢） | **不含任何“协调大模型”**，完全由确定性代码实现。维护任务状态机、管理 Owner 级 FIFO 队列、路由后台权限审批（Preview-and-Commit）、处理跨连接状态恢复，并在“安全插入窗口”将完成的任务结果择时回流至前台播报。 | `server/src/orchestration/`、`server/src/task/`、`server/src/voice/realtime-task-presentation.mjs` |
| **第三层：Backend Agent（后台执行体）** | **负责“干”**（独立上下文长耗时执行） | 接收自包含的目标描述（`objective`），在独立的工具箱、工作区、MCP 服务和 Skills 环境中完成多步规划、检索、编码或复杂业务流。通过统一的 `BackendPort` 接口与 ACP / A2A 协议适配器接入。 | `server/src/backend/`、`server/src/backend/adapters/` |
| **Gateway（服务宿主网关）** | **承载三层的“机房”** | 负责把逻辑三层装配起来，提供 WebSocket / WebRTC 监听、鉴权、连接生命周期管理与统一协议入口（Gateway Client Protocol）。 | `server/`、`packages/webrtc/` |
| **Clients（多端客户端）** | **I/O 采集与状态呈现** | 负责麦克风采集、回声消除（AEC）、音频播放、唤醒词检测（如“你好千问”）、视觉帧捕获及 UI 渲染。支持桌面悬浮球（Desktop Orb）、WebUI、终端 TUI、手机端、车载中控屏及 ESP32 硬件卡片。 | `desktop/`、`web/`、`tui/`、`mobile/`、`examples/` |

### 1.2 单向依赖铁律
系统在工程实现上严格遵守**单向向内收敛原则**：前台工具绝不引用后台适配器；Gateway 既不持有具体 UI 组件，也不直接执行终端业务操作。这种强边界设计使得你可以随时替换语音大模型、更换后台 Agent 或定制车载 HMI 客户端，而无需改动核心运行时代码（[三层架构源码剖析](https://blog.csdn.net/gitblog_00133/article/details/166903254)）。

---

## 二、 核心技术机制深度剖析（支撑 AI 助手与座舱的五大支柱）

### 2.1 刻意收窄的前台与非阻塞异步委派（`spawn_thinking`）
为了让前台语音通道永远保持毫秒级响应、不被复杂的推理和工具链拖慢，[Qwen-Audio-Agent](https://github.com/QwenAudio/qwen-audio-agent) 对实时前台进行了近乎苛刻的减法——在默认通用配置下，前台（`server/src/voice/frontend-tools.mjs`）仅暴露 **8 个核心内置工具**（[架构解析文档](https://www.nxrte.com/jishu/71113.html)）：

| 前台内置工具名称 | 功能说明 | 同步/异步语义 |
| :--- | :--- | :--- |
| `spawn_thinking` | 将耗时或多步执行的任务打包为自包含的 `objective` 移交后台 | **纯异步非阻塞**：受理瞬间立即返回 `accepted` 与任务 ID，前台当即开口回应用户 |
| `get_agent_task_status` | 查询后台正在执行或已排队任务的状态与实时进度 | 高优先级控制流查询（插队在当前协调轮之后、普通排队任务之前） |
| `cancel_agent_task` | 请求取消仍在运行或排队中的后台工作 | **确认式取消**：状态先置为 `cancelling`，直至后台确认停止才落定为 `cancelled` |
| `respond_agent_permission` | 将用户对敏感操作（如支付、改签、删库、座舱购买）的口头授权转达给挂起的后台任务 | 精确绑定至发起请求的源任务 ID，恢复挂起的执行流 |
| `schedule_reminder` | 创建定时提醒或周期性任务 | 持久计划任务，即使 Gateway 重启也会将逾期未播报提醒还原补播 |
| `get_current_time` | 获取本地时区当前精确时间 | 轻量本地同步调用 |
| `memory` | 精确读写本地个性化文件 `USER.md`（偏好）与 `MEMORY.md`（事实） | 轻量本地乐观锁读写（无权修改 `ASSISTANT.md` 人设） |
| `notes` | 管理用户命名的待办清单与便签（增删查改） | 轻量本地状态同步 |

> **关键设计——“协调信封（Coordination Envelope）”与保守解读**：
> 当调用 `spawn_thinking` 时，系统通过 `coordinator.mjs` 生成结构化信封，将 `final_asr`（用户本轮语音转写原话，作为唯一事实来源）与 `objective`（前台对意图的保守整理）一并递交后台。前台只负责**意图交接**，绝不越权替后台制定执行计划，也不把冗长的前台闲聊历史塞给后台，从而彻底隔离了前后台上下文污染（[架构解析文档](https://www.nxrte.com/jishu/71113.html)）。

### 2.2 任务状态机与“两个核心解耦”
在 `server/src/orchestration/task-operations.mjs` 中，每个委派任务都遵循严格的生命周期状态机（[Qwen-Audio-Agent 技术报告](https://arxiv.org/html/2609.25195v1)）：
`queued`（排队） $\rightarrow$ `running`（协调运行） $\rightarrow$ `delegated`（已委派第三层独立长任务） $\rightarrow$ `finalizing`（结果整理） $\rightarrow$ `completed` / `failed` / `cancelling` $\rightarrow$ `cancelled`。

这套机制实现了对语音助手至关重要的**两个解耦**：
1. **语音打断（Speech Interruption）与任务取消（Task Cancellation）解耦**：当助手正在说话而用户开口插话时，仅停止当前语音播报，后台正在跑的导航规划、代码编译或深度检索**继续在后台运行**。只有用户明确说“算了，别查了”，前台才会调用 `cancel_agent_task` 终止后台任务。
2. **执行完成（Execution Completion）与结果投递（Result Delivery）解耦（择时回流纪律）**：
   - **让路当前对话**：如果后台任务完成时，用户正在说话或前台正在回答另一个问题，Orchestration Runtime 会将结果暂存在收集窗口（Collection Window）中，绝不贸然打断用户；若同一窗口期内多个任务完成，还会自动合并为一次播报请求。
   - **播完才算数**：结果注入实时前台上下文后，必须等到**语音真正播放完毕**才标记为“已投递（Delivered）”。如果播报中途被用户打断或网络闪断，投递记录支持安全重试，而无需让后台重新跑一遍任务（[架构解析文档](https://www.nxrte.com/jishu/71113.html)）。

### 2.3 环境感知与统一事件协议（Unified Event Protocol）
这是**智能座舱**与**多模态 AI 助手**最核心的机制（详见 [技术报告 2.3 节](https://arxiv.org/html/2609.25195v1)）。
在真实座舱或桌面环境中，世界的状态不仅随语音改变，还会随触控屏操作、车辆传感器变化或摄像头视觉输入而改变。Qwen-Audio-Agent 定义了独立于模型供应商的**统一事件协议**：
- 每个环境事件包含：**数据结构（Schema）**、**保留规则（Retention Rule）** 和 **响应策略（Response Policy）**。
- 当客户端上报事件时，运行时验证后可执行四种策略之一：
  1. **直接处理（Handle Directly）**：运行时内部消化状态变更。
  2. **静默更新前台上下文（Update Context Silently）**：例如副驾在中控屏上手动修改了导航目的地，或调低了空调温度，客户端发送事件静默更新前台上下文，**不触发语音播报**；但当驾驶员下一秒问“到那儿还要多久？”或“怎么还是这么热？”时，前台模型已精确知晓最新状态。
  3. **计划响应（Schedule a Response）**：如低电量预警、胎压异常或后台预约成功，排队等待安全窗口主动开口提醒。
  4. **立即打断并重绘响应（Interrupt & Request New Response）**：针对高优紧急状态变更。

### 2.4 四层上下文与跨会话持久记忆（Memory & VoiceMem 扩展）
系统将前台上下文拆分为四层权限分明的本地 Markdown 文档（[架构解析文档](https://www.nxrte.com/jishu/71113.html)）：
1. `PROMPT.md`：随框架分发的核心交互与路由规则（只读）。
2. `ASSISTANT.md`：AI 助手的身份、名称与语气画像（模型无权自改，且不影响工具路由与事实判断）。
3. `USER.md`：用户的长期偏好（如“开车时喜欢听爵士乐”、“导航偏好避开高速”）。
4. `MEMORY.md`：跨会话沉淀的长期事实与背景知识。

- **乐观锁精确改写与无感沉淀**：模型修改记忆时必须通过版本号校验与唯一定位替换（`old_text` 必须精确匹配一次）；在每次语音会话结束断开连接时，后台抽取器会在防抖、过滤敏感信息、核验用户原话证据的前提下，异步将稳定偏好沉淀至本地文件，绝不阻塞实时对话。
- **可选插件扩展**：除默认轻量 Markdown 记忆外，仓库在 `examples/` 中还提供了 **[VoiceMem](https://qingkeai.online/blog/VoiceMem)**（左右脑双图谱流式记忆，利用 VAD 尾音窗口在 134ms 内完成信息左脑与情感右脑联合检索）和 **LightRAG**（通过通用 `KnowledgeProvider` 接入外部知识库）的集成方案（[GitHub Releases v2.0.0](https://github.com/QwenAudio/qwen-audio-agent/releases)）。

### 2.5 响应守卫（Response Guard）：“说到就要做到”
针对语音大模型常见的“幻觉应答”——即模型嘴上回答了“好的，我马上帮您查一下”，但本轮实际上漏调了 `spawn_thinking` 或业务工具——系统在响应结束处内置了**响应守卫**。一旦检测到 40 字以内以“我来/马上/这就”开头接执行类动词却未触发任何工具调用的短句，守卫会要求模型就同一轮重新判断一次，极大提升了车载与助手场景的执行可靠性（[架构解析文档](https://www.nxrte.com/jishu/71113.html)）。

---

## 三、 深度聚焦：在“智能座舱（Smart Cockpit）”与“AI 助手”中的落地分析

在 [Qwen-Audio-Agent 技术报告（arXiv:2609.25195）](https://arxiv.org/html/2609.25195v1) 的第 3 节与第 4 节中，官方团队专门将**智能座舱（Intelligent Cockpits）**作为头号核心评测与落地标杆场景进行了深度验证。

### 3.1 智能座舱参考架构：共享领域服务（Shared Domain Service）
在车载场景下，存在两个截然不同的时间尺度与三条并行的交互链路：
- **双时间尺度**：驾驶员既需要毫秒级响应的**即时车控/导航指令**（如“打开主驾座椅通风”、“把空调调到 22 度”、“导航去虹桥机场”），又会在行车途中发起**多步长耗时任务**（如“帮我对比附近三家商场哪家有儿童乐园且停车便宜，顺便买两张今晚的电影票”）。
- **三路状态并发变更**：车辆与座舱应用状态可能被 **① 前台语音直接工具**、**② 后台 Agent 工具** 或 **③ 驾乘人员直接触摸车载 GUI 屏幕** 修改。

**Qwen-Audio-Agent 的座舱解法**：
1. **共享业务状态层（Shared Domain Service）**：前台工具、后台工具与车载 GUI 界面共用同一个领域服务（Domain Service）和业务状态机。无论通过语音还是触屏改变了车窗、空调、媒体或导航状态，都会转化为**统一环境事件（Environmental Events）**回流至 Orchestration Runtime，确保后续每一句语音对话都牢牢锚定在当前真实的车辆状态上。
2. **前后台工具路由策略（混合执行 Mixed Execution）**：
   - **前台直接挂载（Direct Frontend Tools）**：车窗/空调/座椅/灯光控制、基础导航切换、音乐播放控制等短链路、确定性、低延迟工具。
   - **后台异步委派（Backend Delegated Tasks）**：多条件地点检索与比选、车载点餐/购物、长途充电路线多步规划、深度问答等需要多轮工具调用和推理的任务。
   - **安全审批闭环（Preview-and-Commit）**：遇到涉及资金支付（点餐买票）或高风险修改的操作，后台工具先生成预览（Preview）并挂起任务，通过 Orchestration Runtime 交由前台语音向驾驶员播报确认（“已为您选好两张 19:30 的电影票共 88 元，确认支付吗？”），驾驶员口头回答“确认”后，前台调用 `respond_agent_permission` 唤醒后台完成最终提交（Commit）。

### 3.2 座舱基准评测数据：为什么必须采用“前后台混合执行”？
官方在包含 **134 个座舱真实用例（86 个短交互 + 48 个多步复杂任务，共 159 个对话轮次、41 个车载业务工具）** 的内部座舱基准（Cockpit Benchmark）上，使用 `Qwen Audio 3.0 Realtime Plus` 作为前台、`qwen3.8-max` 作为后台，对比了四种架构配置（数据源自 [Qwen-Audio-Agent 技术报告 Table 2](https://arxiv.org/html/2609.25195v1)）：

| 架构执行配置 (Configuration) | 工具调用 F1 | 参数准确率 (%) | 整体任务成功率 (%) | 回复质量 (%) | 短任务成功数 (共86) | 多步任务成功数 (共48) | 短任务平均延迟 (s) ↓ | 多步任务平均延迟 (s) ↓ | 整体平均延迟 (s) ↓ |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **纯前台直连 (Direct tools)** | 0.947 | 93.6% | 72.39% | 95.5% | 83 / 86 | 14 / 48 | **1.488s** | 67.705s | 6.455s |
| **纯后台全委派 (All delegated)** | 0.908 | 89.8% | 80.60% | 89.6% | 71 / 86 | 37 / 48 | 4.266s | **38.653s** | 6.845s |
| **纯文本后台基线 (Text baseline)** | **0.963** | 94.1% | 90.30% | **100.0%** | — | — | 3.149s | 35.449s | 5.572s |
| **前后台混合执行 (Mixed execution)** | 0.962 | **95.2%** | **91.04%** | 97.8% | **84 / 86** | **38 / 48** | 1.591s | 43.430s | **4.729s** |

**对座舱与 AI 助手架构师的核心启示**：
1. **单靠端到端语音大模型直调工具（Direct tools）走不通复杂场景**：虽然短指令延迟极低（1.488s），但在 48 个多步座舱任务中仅成功 14 个（成功率暴跌，且耗时高达 67.7s），因为实时语音模型既要维持音频流又要做多步推理，极易顾此失彼。
2. **把所有工具都扔给后台 Agent（All delegated）会毁掉基础车控体验**：多步任务成功率虽提升至 37/48，但开关空调、切歌等简单指令的平均延迟从 1.48s 恶化到 4.26s，且短任务成功率反而下降（71/86）。
3. **混合执行（Mixed Execution）是黄金架构范式**：通过前台直调短任务 + 后台异步委派长任务，不仅拿下了最高的 **91.04% 整体任务成功率**（超越纯文本基线的 90.30%），还将整体平均执行延迟降至 **4.729 秒**（比纯前台降低 **26.73%**，比全委派降低 **30.91%**）。

### 3.3 通用 AI 助手与智能硬件扩展场景
除了智能座舱，该仓库还原生支持并提供了多个 AI 助手形态的落地方案（[GitHub README](https://github.com/QwenAudio/qwen-audio-agent)）：
- **桌面全能 AI 助手（Desktop Orb / TUI / WebUI）**：内置常驻悬浮球，支持本地语音唤醒（“你好千问”）、空闲休眠、自定义宠物皮肤状态联动，一边陪用户语音讨论方案，一边在后台调度 `Claude Code`、`Qwen Code`、`OpenClaw` 或 `Codex` 读写代码库、整理文档，完成后将生成的报告或文件引用直接推回对话面板。
- **视觉多模态助手（X-Omni）**：接入 `Qwen3.5-Omni Realtime`、`Qwen3.8 Omni Flash Realtime` 或 `Gemini Live` 等支持视频流的前台模型，实现摄像头画面实时捕获、场景观察与视觉伴随问答。
- **端侧智能硬件助手（AI Passport / 千问语音豆）**：在 `examples/ai-passport` 中提供了基于 **ESP32-C3** 硬件卡片的接入方案（[AI Passport 示例文档](https://github.com/QwenAudio/qwen-audio-agent/blob/main/examples/ai-passport/README_ZH.md)），卡片负责麦克风录音、扬声器播放与屏幕表情动画，通过局域网 `device-relay.mjs`（端口 3101）连接宿主机的 Gateway（端口 18888），让百元级单片机瞬间拥有“实时对话 + 后台重型 Agent”能力。

---

## 四、 可插拔生态矩阵：支持哪些语音模型与后台 Agent？

学习和使用这套架构最大的红利在于**前后台完全解耦、双向可插拔**（[GitHub README](https://github.com/QwenAudio/qwen-audio-agent)）：

### 4.1 实时语音前台（Voice Frontends）支持列表
只需实现统一的 `Realtime Provider` 接口，即可无缝切换云端或纯本地语音链路：

| 语音前台模型 / 方案 | 部署方式 | 核心特性与输入模态 | 适用场景建议 |
| :--- | :---: | :--- | :--- |
| **Qwen Audio 3.1 Realtime Plus**（默认） / **3.0 Realtime** | 云端（百炼 API） | 全双工语音、Smart Turn v2 智能轮次检测、原生工具调用 | 生产级中文/多语种 AI 助手、智能座舱主力前台 |
| **Qwen3.5-Omni / Qwen3.8 Omni Flash Realtime** | 云端（百炼 API） | 全双工语音 + **实时视频流输入（Live Video）** | 带舱内 DMS/OMS 摄像头或行车记录仪视觉感知的座舱助手 |
| **Google Gemini Live / OpenAI GPT-Live** | 云端 API | 跨海外生态的全双工语音及多模态交互 | 海外车型座舱出海方案、跨国多模型对比验证 |
| **Doubao Seeduplex 3.0 / StepAudio 3 Realtime** | 云端 API | 火山引擎 / 阶跃星辰全双工实时语音接入 | 国内多供应商冗余备份与音色风格定制 |
| **Hugging Face Speech-to-Speech** | **纯本地部署** | 本地组装 VAD + STT + LLM + TTS 为兼容接口 | 车端离线弱网兜底、高隐私涉密环境 AI 助手 |
| **MiniCPM-o 4.5** | 本地或云端 | 支持实时视频流输入（无原生工具调用，可走纯对话） | 端侧算力受限设备的轻量化多模态交互 |

### 4.2 后台执行体（Backend Agents）支持列表
通过标准 **ACP（Agent Client Protocol）**、**A2A（Agent2Agent）** 或内置适配器接入，直接复用后台已有的 MCP Server、Skills 和认证配置：
- **原生 ACP 五星推荐（★★★★★）**：`Qwen Code`、`OpenCode`、`OpenClaw`（内置 ACP 桥接）、`Qoder`、`Kimi Code`。
- **原生 / 外部适配器四星支持（★★★★☆）**：`Claude Code`（默认通过 `claude-agent-acp` 适配）、`Codex`、`DeepSeek Harness`、`MiniMax Code`、`Hermes`、`CodeBuddy`、`Pi`。
- **轻量 / 无后台模式**：`None`（纯前台模式，适合仅需语音聊天和简单车控的轻量场景）、`Bare LLM`（直接对接 OpenAI 兼容的 Chat Completions 接口）。

---

## 五、 实操上手指南：如何快速安装、运行与跑通智能座舱 Demo？

### 5.1 环境准备与一键安装
确保本地开发环境满足 **Node.js `22.22.2+` 或 `24.15.0+`**，以及 **npm `10+`**（[安装文档](https://github.com/QwenAudio/qwen-audio-agent/blob/main/docs/getting-started/install.md)）。

**方式一：通过 npm 全局安装（最快体验 AI 助手）**
```bash
npm install -g qwen-audio-agent
```

**方式二：克隆源码安装（推荐！学习架构与运行智能座舱 Demo 必选）**
```bash
git clone https://github.com/QwenAudio/qwen-audio-agent.git
cd qwen-audio-agent
npm install
```

### 5.2 配置通用 AI 助手并启动
1. **生成配置文件**：
   ```bash
   qwenaudio config
   ```
2. **填写环境变量配置（`.env`）**：
   前往阿里云百炼平台获取 API Key，配置前台语音模型与后台 Agent（[快速开始指南](https://github.com/QwenAudio/qwen-audio-agent/blob/main/docs/getting-started/quickstart.md)）：
   ```env
   # 必填：百炼实时语音前台 API Key
   QWEN_AUDIO_REALTIME_API_KEY=your-dashscope-api-key

   # 选填：语音前台模型，默认使用 Qwen Audio 3.1 Realtime Plus
   QWEN_AUDIO_REALTIME_MODEL=qwen-audio-3.1-realtime-plus

   # 选填：后台 Agent 协议（如 openclaw / opencode / qwen-code，设为 none 则为纯前台模式）
   AGENT_PROTOCOL=openclaw

   # 选填：指定后台大模型（留空则直接复用后台 Agent 自身的配置）
   QWEN_AUDIO_AGENT_BACKEND_MODEL=qwen3.7-max
   ```
3. **启动服务与客户端**：
   ```bash
   # 终端 1：启动核心 Gateway 服务（默认监听 127.0.0.1:18888）
   qwenaudio

   # 终端 2：启动终端 TUI 界面（或运行 qwenaudio webui 打开浏览器界面）
   qwenaudio tui
   ```
   *如果你想体验带语音唤醒和悬浮球的桌面 AI 助手，可在源码根目录下编译运行桌面端：`npm run desktop:build:local`（macOS）/ `npm run desktop:build:win`（Windows）/ `npm run desktop:build:linux`（Linux）。*

### 5.3 重点实操：一键跑通官方“智能座舱（Smart Cockpit）”示例
仓库在 `examples/smart-cockpit` 目录下内置了一套完整的智能座舱参考实现（包含车载领域服务 `service`、后台 `agent`、`gateway` 以及模拟车载中控屏的 `client`）（[工程配置参考](https://github.com/cskwork/claude-voice-stack)）：

```bash
# 1. 进入克隆好的仓库根目录，复制智能座舱专属环境变量模板
cp examples/smart-cockpit/.env.example examples/smart-cockpit/.env.local

# 2. 编辑 examples/smart-cockpit/.env.local，填入你的百炼 API Key

# 3. 安装智能座舱示例的依赖
npm run example:smart-cockpit:install

# 4. 一键拉起全套座舱服务（Domain Service + Backend Agent + Gateway + 车载中控 GUI Client）
npm run example:smart-cockpit
```
**启动后建议重点体验并观察以下三条交互链路**：
1. **秒级车控体验**：直接对麦克风说“把空调调到 20 度，打开座椅加热”，观察前台模型如何不经过 `spawn_thinking`，直接调用前台车控工具并在 1.5 秒内刷新中控屏 UI。
2. **边聊边干（多步后台任务 + 并发车控）**：发布一个复杂指令（如让后台搜索周边服务或规划复杂行程），在后台任务处于 `running` 状态时，继续用语音调节车窗或闲聊，观察任务完成后如何择时回流播报。
3. **中控屏手动触控与语音上下文同步**：直接用鼠标点击模拟中控屏修改车辆状态或导航设置，观察客户端如何通过统一事件协议静默更新前台上下文，随后用语音追问该状态，验证“屏音一致”。

---

## 六、 面向 AI 助手与智能座舱开发者的四阶段学习与二次开发路线图

如果你希望吃透这套架构，并将其改造应用于自己的**智能座舱项目**或**垂类 AI 助手产品**，建议按照以下四个阶段循序渐进阅读源码并动手改造：

### 阶段一：建立全局认知与跑通基准链路（Day 1 - Day 2）
- **必读文档与论文**：
  1. 精读仓库内的架构总览 `docs/architecture/overview.zh.md` 与边界约束 `docs/architecture/deep-dive.zh.md`（[三层架构源码剖析](https://blog.csdn.net/gitblog_00133/article/details/166903254)）。
  2. 阅读 [Qwen-Audio-Agent 技术报告（arXiv:2609.25195）](https://arxiv.org/html/2609.25195v1) 的第 2 节（系统架构）与第 3.2 节（智能座舱应用）。
- **动手目标**：在本地跑通 `examples/smart-cockpit` 和 `qwenaudio webui`，打开浏览器开发者工具观察客户端与 Gateway 之间的 WebSocket 消息流（Gateway Client Protocol）。

### 阶段二：精读核心调度与前台语音源码（Day 3 - Day 5）
按以下顺序深入 `server/src/` 目录，掌握“边说边干”的工程实现细节：
1. **前台人设与工具边界**：
   - 阅读 `config/frontend-agent/PROMPT.md`：学习官方是如何用 Prompt 严格约束实时语音模型“什么该自己答、什么该调 `spawn_thinking`、什么绝不能编造”的。
   - 阅读 `server/src/voice/frontend-tools.mjs`：查看 8 个核心工具的 JSON Schema 定义与非阻塞返回逻辑。
2. **确定性编排状态机（核心精髓）**：
   - 阅读 `server/src/orchestration/task-operations.mjs`：理解任务提交、Owner 级 FIFO 队列、状态流转（`queued` $\rightarrow$ `running` $\rightarrow$ `delegated` $\rightarrow$ `completed`）以及 `cancelling` 确认式取消的实现。
   - 阅读 `server/src/orchestration/session-task-coordinator.mjs` 与 `server/src/voice/realtime-task-presentation.mjs`：重点看**择时回流（Safe Insertion Window）**、**租约锁（Claim）**以及**响应守卫（Response Guard）**是如何用纯代码实现的。

### 阶段三：智能座舱与垂类助手定制化改造（Day 6 - Day 10）
当你准备将框架迁移到真实车载座舱（如 Android Automotive / QNX + 车端网关）或自定义 AI 助手时，围绕以下四个扩展点进行二次开发：

| 定制化改造维度 | 核心改造位置与接口 | 座舱 / AI 助手落地实践建议 |
| :--- | :--- | :--- |
| **1. 拆分“前台直调车控”与“后台复杂服务”工具集** | 参考 `examples/smart-cockpit` 与 `server/src/voice/frontend-tools.mjs` | 将车载 CAN 总线控制（空调、车窗、座椅、灯光、音量）和基础地图指令注册为 **Frontend Tools**，保证 `<1.5s` 极速响应；将大众点评检索、车主手册故障排查、外卖点单、复杂充电路线规划封装进 **Backend Agent（MCP / Skills）**。 |
| **2. 接入车载 HMI 触控与传感器环境事件** | 客户端统一事件上报接口 + `Orchestration Runtime` 事件策略 | 当驾驶员手动操作中控屏、切换挡位（P/D/R 挡）、或触发 ADAS 告警时，按 Unified Event Protocol 发送事件：普通 UI 操作设为 **静默更新上下文（Silent Context Update）**；安全告警设为 **计划播报或紧急打断播报**。 |
| **3. 接入舱内多模态视觉感知（DMS / OMS / 行车视觉）** | 切换前台为 `Qwen3.5-Omni Realtime` / `Qwen3.8 Omni Flash Realtime` | 利用框架支持的有界摄像头视频帧流（Bounded Camera Frames）能力，将舱内摄像头或车外环视关键帧随音频流一同送入前台，实现“指着窗外建筑问这是什么”或识别乘员手势与疲劳状态。 |
| **4. 打造端云协同 / 弱网离线兜底语音链路** | 实现 `Realtime Provider` 接口（参考 `docs/voice-frontends/speech-to-speech.zh.md`） | 车辆进入地下车库或隧道无网时，利用可插拔 Provider 机制，将前台无缝降级切换至车机端侧部署的本地 `VAD + FunASR/SenseVoice + 端侧小模型 + 本地 TTS` 链路，保障基础车控不断联。 |

### 阶段四：优化长短期车主记忆与车载知识库（进阶）
- **多驾驶员/乘员档案隔离**：利用框架的 `owner` 级会话隔离机制（如 `openclaw:driver_A:backend`），结合车内声纹识别（Speaker ID）或人脸识别切换当前激活的 `USER.md` 与 `MEMORY.md`，实现千人千面的座椅/空调习惯记忆与音乐偏好推荐。
- **接入车载电子手册（RAG）与流式情感记忆**：
  - 参考仓库中的 **LightRAG 示例**，通过 `KnowledgeProvider` 接口挂载整车用户手册、故障码字典与保养政策知识库。
  - 参考 **VoiceMem 示例**，引入双脑流式记忆，不仅记住车主的历史行程事实（左脑），还能捕捉车主在拥堵路况或长途驾驶时的情绪偏好（右脑），在 500ms VAD 窗口内实现零感知时延的贴心陪伴。

---

## 七、 参考资料与核心链接汇总

- **GitHub 开源主仓库**：[QwenAudio/qwen-audio-agent](https://github.com/QwenAudio/qwen-audio-agent)（含完整源码、`docs/` 文档体系与 `examples/` 场景示例）
- **官方文档站点**：[Qwen Audio Agent Documentation](https://qwenaudio.github.io/qwen-audio-agent/)
- **arXiv 官方技术报告**：[Qwen-Audio-Agent Technical Report (arXiv:2609.25195)](https://arxiv.org/abs/2609.25195) / [HTML 全文阅读版](https://arxiv.org/html/2609.25195v1)
- **核心架构与边界深度解析**：[让对话与执行真正并行：Qwen-Audio-Agent 架构解析（阿里语音 AI）](https://www.nxrte.com/jishu/71113.html)
- **三层源码目录映射指南**：[看懂 qwen-audio-agent 架构：Frontend Agent、Orchestration Runtime 与 Backend Agent 的三层设计](https://blog.csdn.net/gitblog_00133/article/details/166903254)
- **智能硬件端侧接入示例**：[AI Passport（ESP32-C3 千问语音豆）客户端示例](https://github.com/QwenAudio/qwen-audio-agent/blob/main/examples/ai-passport/README_ZH.md)
