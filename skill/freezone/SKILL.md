---
name: freezone
description: "Use when the active chat surface is 虾画/Freezone/canvas, or when the user asks about canvas nodes, graph edits, canvas actions, selections, layout, visual boards, or node-based workflows. This skill is a placeholder contract for injected Freezone tools."
compatibility: Requires DRAMACLAW_API_URL, DRAMACLAW_AGENT_TOKEN, DRAMACLAW_PROJECT_ID, DRAMACLAW_CHAT_SURFACE=freezone, and DRAMACLAW_CANVAS_ID for canvas-scoped operations.
requires:
  env: ["DRAMACLAW_AGENT_TOKEN", "DRAMACLAW_API_URL", "DRAMACLAW_PROJECT_ID"]
---

# Freezone 虾画 Skill

## 一句话定位

- 这个 skill 用来让 Hermes 识别“当前是在虾画/Freezone 画布里工作”。
- 虾画当前主链路是：前端注入只读画布上下文，Agent 输出命令，前端执行并保留历史、选中态、确认流程和工具面板。
- 遇到“节点该怎么理解、什么时候该连线、什么时候不该连、视频节点和合成节点怎么分工”这类判断时，先参考 `references/canvas-operation-manual.md`。

## 先判断在做什么

- 用户在问“有哪些 workflow / 工作流 / 工作流技能 / 类型 / 模板 / 方案”，或要规划短剧、广告视频、产品视频、MV 等整体流程时：交给 `workflows` skill，只做规划，不直接改画布。
- 用户已经在具体操作节点，例如创建、修改、移动、连接、布局、删除、打开工具、运行节点动作：进入画布命令模式。
- 只有用户确认工作流计划后，才把计划落成具体画布节点和连线。

## 画布命令模式

- 只要本轮消息包含 `[SUPERTALE_CANVAS_ONTOLOGY_CONTEXT]`、`[SUPERTALE_CANVAS_NODE_REFERENCES]` 或 `[SUPERTALE_CANVAS_CHAT_COMMANDS]`，普通画布编辑就按前端命令模式处理。
- `[SUPERTALE_CANVAS_ONTOLOGY_CONTEXT]` 是当前画布的只读 overview，用来理解已有 nodes、links、slots、actions 和 current selection；不要把它当执行结果。
- `[SUPERTALE_CANVAS_NODE_REFERENCES]` 是本轮明确目标节点。若 overview 和 node references 同时存在，优先以 node references 作为操作目标。
- 节点内的 `action_catalog_json` 是前端操作目录：
  `execution="chat_command"` 用对应 `command_type`；
  `execution="manual_ui"`、`execution="frontend_node"`、`requires_confirmation` 用 `run_node_action`，并使用动作里的 `action` 字段。

## 画布建模原则

- 画布连线表示数据输入、参考或上下文关系，不表示“下一步顺序”。
- `add_next_node` 只在当前节点会作为新节点输入时使用。若只是创建一个相关节点但当前节点不是输入，应使用 `create_node`，必要时再补 `create_edge`。
- 多输入任务应把多个输入节点分别连接到目标节点，而不是串成 `A -> B -> C`。例如“图片 + 文本生成视频”应让图片节点和文本节点分别连接到 `videoNode`。
- 文本作为图片生成输入时，可以使用 `textAnnotationNode -> imageGenNode`；图片节点自身 prompt 会和上游文本共同构成生成上下文。
- `textAnnotationNode` 是默认的普通文本语义节点。人物设定、广告创意、分镜描述、配音稿、普通脚本内容，优先用它。
- `scriptNode` 不是默认文本节点，也不是默认工作流中间节点。只有用户明确要结构化脚本、镜头表、分镜表，或明确要求脚本生成器产物时，再考虑使用它。
- 不要复用历史轮次里的画布节点 ID。只有当前用户文本、当前 node references、或本轮刚创建的 `client_id` 可作为操作目标。

## 视频节点原则

- 用户说“做成视频 / 生成视频 / 做广告短片 / 生成短片 / 素材都有了怎么做”时，默认应创建 `videoNode`，用于根据上游图片、文本、脚本、音频参考生成一个视频片段。
- `videoNode` 是视频生成节点，不是最终时间线节点。它负责生成单个镜头或单段视频素材。
- `videoNode` 常见模式包括：文生视频、图生视频、首尾帧视频、全能参考、图片参考。是否可切换、真实字段名和枚举值是什么，以当前节点的 `action_catalog_json.editable_schema` 为准；不要猜字段或自造值。
- 当用户要求“改成图生视频 / 改成首尾帧 / 改成全能参考 / 改成图片参考 / 改视频模式”时，优先理解为修改 `videoNode` 的生成模式，而不是修改视频模型、视频比例或把节点换成 `videoComposeNode`。
- 不要默认创建 `videoComposeNode`。`videoComposeNode` 只用于已有一个或多个真实视频片段后做时间线合成/最终成片导出。
- 只有当上游已经有带 `videoUrl` / `video_url` 的 `videoNode`，或用户明确说“时间线合成 / 合成多个视频片段 / 剪成最终成片 / video compose”时，才考虑 `videoComposeNode`。

## 常用命令约定

- 节点 CRUD、布局、打开工具和节点自身动作都走 `canvas_chat_commands.v1`。
- `move_nodes` 用于移动节点。相对移动用 `dx/dy`；移动到指定坐标再用 `positions`。如果本轮引用了多个节点且用户表达的是整体移动，必须把所有目标节点都放进同一个 `node_ids`。
- 删除节点时用 `delete_nodes`，把所有目标节点 ID 放进 `node_ids`。不要因为引用里有边就改成 `delete_edges`。
- 只有用户明确要求“断开连接 / 移除连线 / 解绑连接”时，才使用 `delete_edges`。
- 如果用户要求“执行生成/运行/生成”当前引用的 `imageGenNode`，且 `action_catalog_json.actions` 中存在 `generate_image`，则使用 `run_node_action` 调用该动作。

## 用户可见回复

- 面向用户时称为“虾画”；不要主动解释 Hermes、plugin、toolset、注入块、协议名、schema 名、工具名、字段名或桥接细节。
- 内部实现信息只用于推理或调用工具，不要写进自然语言回复。
- 用户可见回复只说业务动作、业务对象、等待状态和业务结果。
- 不要声称已移动、创建、删除、连接或修改节点。`freezone_emit_canvas_command` 成功只表示命令已提交给前端执行器；最终是否应用成功取决于前端执行结果。
- 当前会话若未注入 `DRAMACLAW_CANVAS_ID`，说明没有绑定具体画布，只能做项目级解释或要求用户先打开一个画布。
- `canvasId` 表示画布 ID，不是节点 ID。

## 工作流列表查询

当用户问“有哪些 workflow / 有哪些工作流 / 支持哪些工作流 / 有哪些工作流技能 / 我的工作流技能有哪些 / 有哪些 workflow skill / 有哪些流程 / 有哪些类型 / 有哪些模板 / 有哪些方案 / 可以创建哪些类型 / 可以做哪些流程”时，这是纯查询请求，必须直接回复：

```text
当前已注册的工作流有 4 个：

1. 短剧 / 小说转视频
2. 广告视频
3. 产品视频 / 商品演示
4. MV / 音乐视频

你可以选择其中一个，我再根据对应类型提示你需要提供哪些材料。
```

不要输出 `freezone_workflow_plan.v1`，不要要求用户提供材料，不要等待确认，不要调用或等待画布命令，不要列未注册的示例类型。

“工作流技能 / workflow skill”默认指 `workflows` skill 中注册的工作流规划能力，不是 `freezone.sketch_from_context`、`freezone.frame_from_context`、`freezone.scene_360` 这类画布节点可执行原子技能。只有用户明确说“画布 skill / 节点技能 / 可执行技能 / action / action_catalog / 生成草图技能 / 分镜技能”时，才列画布 action catalog。

## 工作流规划

当用户选择或要求创建某类工作流时，交给 `workflows` skill，由它按意图读取对应 reference：

- 短剧、小说转视频、故事转视频：`workflows/references/short-drama.md`
- 广告视频、投放素材、Hook、A/B 版本：`workflows/references/ad-video.md`
- 产品视频、商品演示、功能讲解：`workflows/references/product-video.md`
- MV、音乐视频、歌词视觉化：`workflows/references/mv.md`

所有工作流计划都必须输出 `freezone_workflow_plan.v1`，且只规划，不直接改画布。

## 工具契约

虾画 Agent 的主链路只保留两个工具：

- `freezone_emit_canvas_command`：写画布或操作前端 UI。用于发出 `canvas_chat_commands.v1`，由前端执行器应用创建、修改、移动、连接、删除、布局、打开工具、运行节点动作等画布变化。
- `freezone_request_canvas_context`：读画布上下文或动态参数。用于发出 `canvas_context_request.v1`，向前端请求 `canvas_summary`、`node_detail`、`neighbor_graph`、`action_catalog`、`node_create_schema`、`audio_voice_options`、`selection_detail` 等只读信息；它不修改画布，也不需要用户确认。
如果主链路工具返回 `not_configured`、`not_implemented` 或 `canvas_id is required`，直接简短说明当前虾画工具尚未完成注入或未绑定画布，不要改用 shell、curl、文件读写或猜测本地状态。

## 回复规则

- 画布读取类请求：如果没有前端注入的节点引用，才读取当前画布快照再回答；如果已有 `[SUPERTALE_CANVAS_NODE_REFERENCES]`，优先使用注入内容。
- 上下文补充类请求：如果当前注入内容不足以判断合法节点、上下游关系、创建参数、模型/音色/模板选项，优先调用 `freezone_request_canvas_context`；不要用 `freezone_emit_canvas_command` 包装这些读取请求。
- 工作流类请求：如果用户询问有哪些 workflow / 工作流 / 工作流技能 / workflow skill / 流程 / 类型 / 模板 / 方案，使用 `workflows` 回复已注册列表；如果用户意图是规划一条由多个节点、连线和执行阶段组成的生成工作流，使用 `workflows` 分析和计划，不能直接创建节点。用户确认后再进入画布命令模式。
- 画布节点创建、移动、修改、连线、布局、删除、打开工具类请求：如果前端注入了 `[SUPERTALE_CANVAS_ONTOLOGY_CONTEXT]`、`[SUPERTALE_CANVAS_NODE_REFERENCES]` 或 `[SUPERTALE_CANVAS_CHAT_COMMANDS]`，优先调用 `freezone_emit_canvas_command`，由前端执行器应用。
- 全局画布请求：如果用户说“看看画布”“整理当前画布”“基于现有节点继续做”，且不是主线工作流规划意图，优先读取 `[SUPERTALE_CANVAS_ONTOLOGY_CONTEXT]` 中的 objects/links/current_selection，选择已有节点或用 `create_node`/`create_edge`/`layout_nodes` 等前端命令落地。
- 不展示原始 API URL、认证头、文件系统路径、内部字段名、工具名或内部 JSON，除非用户明确要求调试接口契约。
