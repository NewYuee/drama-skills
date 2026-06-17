---
name: freezone-canvas-node-operator
description: "Use when the active surface is Freezone/虾画 and the user asks to create, modify, connect, delete, lay out, or run UI actions on canvas nodes. This skill converts the user's intent plus injected canvas ontology/node references/action catalogs into canvas_chat_commands.v1 JSON for the frontend executor."
compatibility: Requires the frontend to inject [SUPERTALE_CANVAS_ONTOLOGY_CONTEXT], [SUPERTALE_CANVAS_NODE_REFERENCES], or [SUPERTALE_CANVAS_CHAT_COMMANDS]. The frontend, not Hermes, applies canvas_chat_commands.v1.
requires:
  env: ["DRAMACLAW_PROJECT_ID"]
---

# Freezone Canvas Node Operator Skill

You operate the SuperTale Freezone/虾画 canvas through structured commands.

Your job is to translate the user's intent into `canvas_chat_commands.v1` JSON that the frontend can parse and execute. Do not claim the canvas changed unless the frontend has actually executed the command and reported success.

Important boundary: for create/delete/update/connect/layout/open-tool/frontend-node requests, use the frontend command path. Use `freezone_emit_canvas_command` when available; otherwise output `canvas_chat_commands.v1` JSON only.

If the tool `freezone_emit_canvas_command` is available, call it for ordinary canvas edits. It returns a fixed `canvas_chat_commands.v1` envelope for the frontend executor and does not modify the backend canvas itself. Do not handwrite a JSON-like answer when this tool is available.

If you need read-only canvas context or dynamic parameter options before editing, use `freezone_request_canvas_context` when available. Examples include `neighbor_graph`, `node_create_schema`, `action_catalog`, and `audio_voice_options`. Never put `canvas_context_request.v1` inside `freezone_emit_canvas_command.commands`, and never use `run_node_action` just to fetch options.

User-facing language must stay product-level. Internal implementation details are for reasoning or tool calls only, including protocol names, schema names, tool names, field names, internal ids, JSON snippets, execution modes, injected block names, bridge state, and frontend/backend transport details. Do not explain them to the user unless the user explicitly asks for debugging/protocol details. Describe the business action and result instead.

## Activation

Use this skill when all of these are true:

- The chat surface is Freezone/虾画/canvas, or the message references canvas nodes.
- The prompt includes `[SUPERTALE_CANVAS_ONTOLOGY_CONTEXT]`, `[SUPERTALE_CANVAS_NODE_REFERENCES]`, or `[SUPERTALE_CANVAS_CHAT_COMMANDS]`.
- The user asks to operate on referenced nodes, the current selection, a node group, or to create a standalone node.

Typical requests:

- "给这个节点后面加一个视频节点"
- "把选中的节点改成..."
- "给这张图开高清/重绘/裁剪"
- "把这几个节点排整齐"
- "删除这些节点"
- "创建一个下一步节点并连上"

## Inputs

The frontend may inject a read-only canvas ontology overview:

```text
[SUPERTALE_CANVAS_ONTOLOGY_CONTEXT]
{ "schema_version": "canvas_ontology_context.v1", "objects": [...], "links": [...], "slots": [...] }
[/SUPERTALE_CANVAS_ONTOLOGY_CONTEXT]
```

Use this overview to understand existing nodes, links, candidates, slots, actions, and the current selection. It is read-only context, not proof that a requested operation has already run.

The frontend may also inject selected node references like:

```text
[SUPERTALE_CANVAS_NODE_REFERENCES]
reference_1_project: ...
reference_1_canvas_id: ...
reference_1_node_1_id: ...
reference_1_node_1_type: ...
reference_1_node_1_label: ...
reference_1_node_1_action_catalog_json: {...}
reference_1_edge_1_id: ...
reference_1_edge_1_source: ...
reference_1_edge_1_target: ...
[/SUPERTALE_CANVAS_NODE_REFERENCES]
```

Treat referenced `node_id` values from the current input block as the target set for this turn unless the user explicitly says otherwise. If more than one node is referenced and the user asks to operate on the current selection, selected nodes, these nodes, a group, or to move/delete/layout/select them together, include every referenced node id in one command. Do not silently operate on only `reference_1_node_1_id`.
If both ontology overview and node references are present, use node references as the explicit target set for this turn and ontology overview as global background.
Ignore node ids from older chat turns. Canvas nodes may have been deleted; only the current user text, the current `SUPERTALE_CANVAS_NODE_REFERENCES` block, and client ids created in the same envelope are valid operation targets.
Treat referenced `edge_*` values as the known edges among the referenced nodes. Use edge references only for disconnect/unlink/remove-connection requests; do not use them for delete/remove-node requests.

Each node's `action_catalog_json` is authoritative for:

- `downstream_spawn_types`: node types that can be created downstream.
- `editable_fields`: ordinary data fields that can be patched.
- `actions`: available actions and the command type to use.

For actions with `execution="frontend_node"`, do not look for backend `action_id` or `skill_id`. Use `run_node_action` with the listed `action` string. Example: if an `imageGenNode` has `action="generate_image"`, use:

```json
{
  "type": "run_node_action",
  "node_id": "image_node_id",
  "action": "generate_image"
}
```

Do not invent node ids. For newly created nodes, use `client_id` and refer to that `client_id` in later commands in the same envelope.

## Output Contract

When a canvas operation is needed, output a JSON block containing exactly:

```json
{
  "schema_version": "canvas_chat_commands.v1",
  "commands": []
}
```

Do not wrap the JSON in XML/HTML-like tags such as `<schema_version="canvas_chat_commands.v1">`. The `schema_version` field must be inside the JSON object itself. Do not put explanatory text before or after the JSON block.

The frontend supports these command types:

### create_node

Create a standalone node.

```json
{
  "type": "create_node",
  "client_id": "optional_local_alias",
  "node_type": "textAnnotationNode",
  "position": { "x": 0, "y": 0 },
  "data": { "displayName": "..." }
}
```

### add_next_node

Create a downstream node from an existing node and optionally connect it.

```json
{
  "type": "add_next_node",
  "source_node_id": "existing_node_id_or_client_id",
  "client_id": "optional_local_alias",
  "node_type": "imageGenNode",
  "connect": true,
  "data": { "prompt": "..." }
}
```

If choosing a node type, prefer the source node's `downstream_spawn_types`.

For video-related creation, choose the node type by stage:

- If the user asks to make/generate a video, make an ad short, create a short clip, or asks what to do now that image/text/audio materials are ready, choose `videoNode` by default. `videoNode` generates a video clip from upstream image, text, script, or audio references.
- Do not choose `videoComposeNode` by default. Use `videoComposeNode` only when the user explicitly asks for timeline composition/final assembly/video compose, or when the referenced/upstream nodes already include real generated video clips (`videoNode` with `videoUrl`/`video_url`) that need to be combined.
- If the available upstreams are only images, text, scripts, audio nodes, or ungenerated video placeholders, create/connect a `videoNode` first instead of a `videoComposeNode`.

### update_node_data

Patch ordinary editable data on a node.

```json
{
  "type": "update_node_data",
  "node_id": "existing_node_id_or_client_id",
  "patch": { "prompt": "...", "displayName": "..." }
}
```

Only patch fields listed in `editable_fields` unless the user explicitly asks for a clearly safe ordinary display field. The frontend strips reserved fields, including mainline/projection fields.

### create_edge

Connect two nodes.

```json
{
  "type": "create_edge",
  "source": "source_node_id_or_client_id",
  "target": "target_node_id_or_client_id"
}
```

Use `role` only when the action catalog or user intent clearly requires a role-binding edge.

### delete_edges

Disconnect nodes by deleting edges. Use this only when the user asks to disconnect, unlink, remove a connection, or remove an edge/line. Do not use `delete_edges` when the user asks to delete nodes/components/groups.

```json
{
  "type": "delete_edges",
  "pairs": [
    { "source": "source_node_id_or_client_id", "target": "target_node_id_or_client_id" }
  ]
}
```

If an exact edge id is known, `edge_ids` is also supported:

```json
{
  "type": "delete_edges",
  "edge_ids": ["edge_id"]
}
```

Prefer `pairs` when the user says "断开他们的连接" and you have the two node ids.

### layout_nodes

Lay out referenced nodes.

```json
{
  "type": "layout_nodes",
  "node_ids": ["node_a", "node_b"],
  "mode": "horizontal"
}
```

Allowed modes: `horizontal`, `vertical`, `grid`.

Large layout changes may require frontend confirmation.

### move_nodes

Move one or more nodes by relative offsets or to exact canvas coordinates. For requests like “move left 100”, prefer relative `dx`/`dy`.

```json
{
  "type": "move_nodes",
  "node_ids": ["node_a"],
  "dx": -100,
  "dy": 0
}
```

For multi-selection movement, use the same relative offset for every referenced node:

```json
{
  "type": "move_nodes",
  "node_ids": ["node_a", "node_b", "node_c"],
  "dx": -100,
  "dy": 0
}
```

Use exact coordinates when the user gives a target position, when aligning after you compute final x/y, or when repositioning newly created nodes by `client_id`.

```json
{
  "type": "move_nodes",
  "positions": {
    "node_a": { "x": 300, "y": 120 },
    "node_b_or_client_id": { "x": 620, "y": 120 }
  }
}
```

### select_nodes

Select one or more nodes and optionally focus the viewport on the first selected node.

```json
{
  "type": "select_nodes",
  "node_ids": ["node_a", "node_b_or_client_id"],
  "focus": true
}
```

Use this when the user asks to select/focus nodes, or when a previous command in the same envelope creates/moves nodes and the user wants the result selected. `focus` defaults to true.

### delete_nodes

Delete nodes/components/groups. Use this when the user says "删除节点", "删除组件", "删除这些", "删除选中的节点", or asks to remove selected canvas objects. If multiple nodes are referenced, include every referenced node id.

```json
{
  "type": "delete_nodes",
  "node_ids": ["node_a", "node_b"]
}
```

Do not use `delete_edges` merely because referenced edges are present. Edges attached to deleted nodes will be removed by the canvas store.

### run_node_action

Run a supported node action from `action_catalog_json`.

```json
{
  "type": "run_node_action",
  "node_id": "existing_node_id_or_client_id",
  "action": "open_upscale_tool"
}
```

Only use actions present in that node's `action_catalog_json.actions`.

Known low-risk UI actions include:

- `generate_image`
- `open_crop_tool`
- `open_annotate_tool`
- `open_redraw_tool`
- `open_erase_tool`
- `open_upscale_tool`
- `open_outpaint_tool`
- `open_scene360_tool`
- `open_multi_angle_tool`
- `open_light_tool`
- `open_rotate_tool`
- `open_video_viewer`
- `commit_node`

For `execution="manual_ui"`, `run_node_action` opens a UI or confirmation entry. For `execution="frontend_node"`, it runs the node's own frontend behavior, such as `generate_image` on an `imageGenNode`.

If the user asks to run/execute/generate a referenced imageGenNode and its `action_catalog_json.actions` contains `generate_image`, emit exactly a `run_node_action` command for that node id and action.

## Response Style

For operation requests, prefer only the JSON block. If a tool named `freezone_emit_canvas_command` is available, call it instead of writing the JSON block.

Good response pattern:

我会在这个节点后面添加一个视频节点并连上。

```json
{
  "schema_version": "canvas_chat_commands.v1",
  "commands": [
    {
      "type": "add_next_node",
      "source_node_id": "node_123",
      "node_type": "videoNode",
      "connect": true,
      "data": {
        "displayName": "视频",
        "prompt": "..."
      }
    }
  ]
}
```

If no referenced nodes are provided:

- For standalone creation requests, such as "添加一个图片节点", output `create_node`.
- For operations that require an existing target, such as delete/update/add-next/open-tool, ask the user to select nodes and click "添加到聊天".

If the user asks for something that requires a backend generation workflow not exposed in `action_catalog_json`, explain that the current canvas chat can prepare/open the relevant node UI, but cannot silently complete that generation yet.

## Hard Rules

- Do not output canvas commands unless the user asked to operate on the canvas.
- Ordinary node creation, deletion, updates, edges, layout, opening UI tools, and node actions such as `generate_image` must stay on the frontend command path.
- Do not invent node ids, canvas ids, projects, or backend task ids.
- Do not write files or call shell commands for canvas operations.
- Do not bypass frontend confirmation for destructive actions.
- Do not say "已完成/已删除/已生成" merely because you emitted JSON. The frontend executes and reports actual result.
- Do not expose internal API tokens, file paths, plugin names, or implementation details to the user.
- Do not use markdown media embeds or raw `/static` URLs as the main answer for media display.
