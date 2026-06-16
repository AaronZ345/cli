# 主题资料收集工作流：审核与计划

由状态 `RESOURCE_RESOLVE`、`CONTENT_VERIFY`、`RELEVANCE_CLASSIFY`、`PLAN_MOVE` 加载。

本文档负责资源解析、内容验证、相关性分级、审核 UI、移动计划生成、`ResourceItem` 和 `MovePlanItem`。不得创建目标、移动资源或执行恢复操作。

本文档只服务 `workspace-topic-move-collector`。进入本文档时，`active_workflow` 必须是 `workspace-topic-move-collector`；不得把当前任务改路由到其他 workflow。

## 必读上下文

执行本文档规则前：

1. 按 [`../../lark-shared/SKILL.md`](../../lark-shared/SKILL.md) 处理身份、认证和权限。
2. 按 [`lark-drive-inspect.md`](lark-drive-inspect.md) 处理 URL / token 解析。
3. 使用 `drive metas batch_query` 补齐 Drive 资源 owner、标题和 URL。
4. 必要时使用 `drive permission.members auth` 读取权限信号；该接口不提供 `full_access` / 移动权限的直接判定，不能把 `manage_public` 等同为可移动。
5. 按 [`../../lark-wiki/references/lark-wiki-node-get.md`](../../lark-wiki/references/lark-wiki-node-get.md) 处理 Wiki 节点解析。
6. 按 [`../../lark-doc/references/lark-doc-fetch.md`](../../lark-doc/references/lark-doc-fetch.md) 读取文档内容。
7. 需要验证 Sheet 内容时，按 [`../../lark-sheets/SKILL.md`](../../lark-sheets/SKILL.md) 执行。

## 进入审核阶段前校验

进入本文档后，如果 `resource_items` 还不存在，当前状态必须是 `RESOURCE_RESOLVE`。

禁止从 `candidate_items` 直接进入 `RELEVANCE_CLASSIFY`。即使候选项已有标题、URL、摘要或 token，也必须先执行 `RESOURCE_RESOLVE`，再执行 `CONTENT_VERIFY`。

## 状态：`RESOURCE_RESOLVE`

进入条件：候选列表已准备。

必须：

1. 将每个 `CandidateItem` 转换为标准化 `ResourceItem`。
2. 解析 canonical token、资源类型、URL、当前父级、Wiki 节点身份和读取权限状态。
3. 对 Wiki 资源同时保留 `wiki_node_token` 和 `wiki_obj_token`。
4. 补齐 `owner_id`、`is_owner`、`move_permission_state`、`move_permission_basis` 和 `target_write_state`。
5. 基于 `target_location` 检测不支持的移动方向。
6. 未解析成功的资源仍保留在审核分组中，不得静默丢弃。
7. 即使搜索结果已经包含标题、URL 或 token，也必须经过本状态生成 `ResourceItem`；不得从召回结果直接进入相关性分级。
8. 只有确认 `move_permission_state=movable` 的资源，才能进入后续默认移动链路。
9. 解析耗时超过约 60 秒时，必须输出进度提示，之后约每 60 秒提示一次。

### 解析规则

| 候选类型 | agent 必须执行 |
|----------------|---------------|
| Drive URL / token | token 或类型不确定时，使用 `drive +inspect`。 |
| Wiki URL / token | 使用 `drive +inspect` 或 `wiki +node-get`；保留节点身份和对象身份。 |
| 文件夹候选 | 标记为容器；不要当作普通文档做内容验证。 |
| 快捷方式候选 | 能解析源资源时解析源资源；同时保留快捷方式身份。 |
| 无读取权限 | 保留可见元数据，并设置 `permission_state=denied`。 |
| 无移动权限或移动权限未知 | 保留可见元数据和召回证据，并设置对应 `move_permission_state`。 |

### 资源解析进度 UI

当 `RESOURCE_RESOLVE` 持续超过约 60 秒时，输出当前进度：

```text
资源解析进度：已解析 <resolved_count>/<total_count> 项，已确认可移动 <movable_count> 项，无移动权限 <denied_count> 项，移动权限未知 <unknown_count> 项，解析失败 <failed_count> 项。
当前资源：<title>
继续解析中，不会创建或移动资源。
```

如果正在处理权限或 owner 元数据，可补充：

```text
当前步骤：解析 owner / 当前父级 / 移动资格。
```

`RESOURCE_RESOLVE` 完成后，输出摘要：

```text
资源解析完成：
- 候选总数：N 项
- 可进入内容验证：N 项
- 无移动权限：N 项
- 移动权限未知：N 项
- 解析失败或无读取权限：N 项

下一步会对可移动资源做内容验证；不会创建或移动资源。
```

### 移动资格判定

`RESOURCE_RESOLVE` 必须按以下顺序判断移动资格：

| 条件 | `move_permission_state` | `move_permission_basis` |
|------|--------------------------|--------------------------|
| 当前用户是资源 owner | `movable` | `owner` |
| 当前上下文有明确 `full_access` / 可管理权限证据 | `movable` | `explicit_full_access` |
| API 明确返回资源侧权限不足 | `denied` | `api_denied` |
| 目标位置确认不可写 | `denied` | `target_denied` |
| 目标方向或资源类型不支持移动 | `denied` | `unsupported_direction` |
| 当前用户不是 owner，且没有明确可管理权限证据 | `unknown` | `not_owner_unverified` |
| 无法取得 owner 或权限元数据 | `unknown` | `metadata_unavailable` |

注意：

1. `owner` 是强证据，但不是唯一证据；`full_access` / 可管理权限也可作为可移动证据。
2. `drive permission.members auth` 不提供 `full_access` 或 `move` action；不能用 `view`、`edit`、`share` 或 `manage_public` 结果推断可移动。
3. 目标位置权限单独写入 `target_write_state`；目标不可写时，不得生成可执行移动计划。
4. `move_permission_state=unknown` 的资源默认不进入内容验证、相关性高 / 中分组或移动计划。
5. 当 `owner_scope=mine` 但解析出的 owner 不是当前用户时，将该资源视为异常候选，标记为 `move_permission_state=unknown`，不得加入移动计划。

## 状态：`CONTENT_VERIFY`

进入条件：资源列表已准备。

必须：

1. 只在资源解析后读取支持的内容。
2. 按数量、大小和类型能力限制读取范围。
3. 结合搜索证据和内容证据；除非标题精确且足够强，否则不要仅凭标题判为高相关。
4. 将不可读取资源标记为 `unverifiable` 或 `permission_denied`。
5. 不得自动申请权限。
6. 为每个资源写入验证状态：已读取内容证据、仅可使用搜索证据、无权限、无移动权限、移动权限未知、无法验证或不支持内容验证。
7. 只有所有资源都有验证状态后，才能进入 `RELEVANCE_CLASSIFY`。
8. 对 `move_permission_state=denied|unknown` 的资源，不再读取正文内容，写入跳过验证原因并保留召回证据。

### 验证方式

| 资源类型 | 验证方式 |
|---------------|---------------------|
| `docx` / `doc` | 允许时使用 `docs +fetch --api-version v2`。 |
| `sheet` | 使用 `sheets +find` 查关键词证据，或用 `sheets +read` 读取有界范围。 |
| `bitable` | 只有必要且已加载 Base 能力时验证。 |
| `slides` | 除非具备幻灯片读取能力，否则使用元数据 / 预览 / 标题证据。 |
| `file` | 仅在支持时使用标题、元数据、预览或导出文本。 |
| `wiki` 节点 | 按 `obj_type` 验证底层对象；节点本身不是内容 token。 |
| `folder` | 除非用户明确要移动容器，否则通常不作为主题证据移动。 |

## 状态：`RELEVANCE_CLASSIFY`

进入条件：内容证据已准备，且每个 `ResourceItem` 都已有验证状态或跳过验证原因。

禁止条件：

1. 只有 `candidate_items`，没有 `resource_items`。
2. 资源未经过 `RESOURCE_RESOLVE`。
3. 资源没有 `RESOURCE_RESOLVE` 写入的移动资格状态。
4. 资源没有 `CONTENT_VERIFY` 写入的验证状态或跳过验证原因。

必须将每个资源归入且只归入一个分组：

| 分组 | 说明 | 默认移动 |
|-------|------|--------------|
| `high` | 可移动资源，且主题或内容直接命中，有明确标题 / 正文 / 表格 / 评论证据。 | 是 |
| `medium` | 可移动资源，可能相关，但证据不足或只命中弱相关片段。 | 否，需用户选择 |
| `low` | 可移动资源，弱相关或噪声，保留展示但不建议移动。 | 否 |
| `permission_denied` | 当前身份无权读取或解析，不能验证内容。 | 否 |
| `no_move_permission` | 已确认当前身份不具备移动资格。 | 否 |
| `move_permission_unknown` | 无法确认当前身份是否具备移动资格。 | 否 |
| `unverifiable` | 类型或工具限制导致无法验证内容。 | 否 |
| `unsupported_move_target` | 目标方向或资源类型不支持移动。 | 否 |

`high`、`medium` 和 `low` 只能包含 `move_permission_state=movable` 的资源。

判为高相关至少需要一个强证据：

1. 标题或内容中出现精确主题短语。
2. 多个主题词在相关上下文中同时出现。
3. Sheet / 表格单元格明确匹配用户主题。
4. 用户明确提供的文档名或项目别名命中。

中相关示例：

1. 标题包含一个主题词，但内容无法确认。
2. 搜索摘要看起来相关，但无法完整读取。
3. 别名命中合理但证据不够强。

## 审核 UI

必须展示每个分组中的资源名称。

默认展示规则：

1. 展开 `high` 和 `medium`。
2. 折叠 `low`、`permission_denied`、`no_move_permission`、`move_permission_unknown`、`unverifiable` 和 `unsupported_move_target`，但展示数量并允许展开。
3. 每个可见资源展示标题、类型、当前位置、证据和默认动作。
4. 除非用户要求技术细节，否则不展示原始 token。

示例：

```text
筛选结果：

搜索范围：<当前用户 owner / 负责的资源 | 所有当前身份可见资源>

高相关（默认移动）：
- 标题｜类型｜证据｜当前位置

中相关（需你勾选后才移动）：
- 标题｜类型｜证据｜当前位置

未默认移动：
- 低相关：N 项
- 无权限：N 项
- 无移动权限：N 项
- 移动权限未知：N 项
- 无法验证：N 项
- 不支持移动：N 项

你可以选择：
1. 确认按默认规则生成移动计划。
2. 勾选要加入计划的中相关资源。
3. 要求把某些资源移到其他分组或从计划中移除。
4. 展开低相关 / 无权限 / 无移动权限 / 移动权限未知 / 无法验证 / 不支持移动分组查看名称。
```

### 用户调整规则

如果用户不同意相关性结果，必须基于用户要求更新 `relevance_groups`，再重新展示分组结果并重新生成后续移动计划。

典型调整包括：

1. 从 `high` 中移除某个资源。
2. 将 `medium` 中某个资源提升为 `high`。
3. 将某个资源标为 `low` 或不移动。
4. 要求重新读取证据或重新判断一批资源。
5. 要求重新确认某些资源的移动权限。

用户调整后：

1. 旧的 `move_plan_items` 立即失效。
2. 必须先输出“调整后相关性结果”，展示被调整项、各分组数量和高 / 中相关资源名称。
3. 不得只回复“已调整”，也不得直接跳到 `CONFIRM_EXECUTION`。
4. 必须基于新的 `relevance_groups` 重新执行 `PLAN_MOVE`。
5. 不得把 `no_move_permission` 或 `move_permission_unknown` 资源直接提升到 `high` / `medium`；必须先回到 `RESOURCE_RESOLVE` 取得可移动证据。

### 调整后结果 UI

```text
已按你的要求调整相关性结果：
- <标题>：<原分组> -> <新分组>

调整后分组：

搜索范围：<当前用户 owner / 负责的资源 | 所有当前身份可见资源>

高相关（默认移动）：N 项
- 标题｜类型｜证据｜当前位置

中相关（需你勾选后才移动）：N 项
- 标题｜类型｜证据｜当前位置

未默认移动：
- 低相关：N 项
- 无权限：N 项
- 无移动权限：N 项
- 移动权限未知：N 项
- 无法验证：N 项
- 不支持移动：N 项

接下来会基于这个调整后的结果重新生成移动计划；你也可以继续调整。
```

## 状态：`PLAN_MOVE`

进入条件：相关性分组已准备。

必须：

1. 当 `target_location.create_required=true` 时，纳入目标创建计划。
2. 默认纳入全部 `high` 且 `move_permission_state=movable` 的资源。
3. 只有用户明确选择时，才纳入 `medium` 且 `move_permission_state=movable` 的资源。
4. 默认排除 `low`、`permission_denied`、`no_move_permission`、`move_permission_unknown`、`unverifiable` 和 `unsupported_move_target`。
5. 为跳过项生成原因。
6. 为每个计划项生成稳定 `plan_id`，供 `rollback_snapshot` 和 `execution_journal` 关联。
7. 按 `move_method` 选择正确 token，不得把 Wiki 底层对象 token 当作 Wiki 节点移动 token。
8. 在计划阶段标记 `rollback_supported` 和 `rollback_blocker`；Drive 文档移动到 Wiki 默认 `rollback_supported=false`。
9. 为执行阶段保留恢复快照所需输入。
10. 停止并等待用户选择或执行意图。
11. 不得为 `move_permission_state=denied|unknown` 的资源生成 `move_resource` 计划项。

### 移动 token 选择

| `move_method` | token 规则 |
|---------------|------------|
| `drive_move` | `source_token` 使用 Drive file / folder token。 |
| `wiki_move_docs_to_wiki` | `source_token` 使用底层 Drive / doc token。 |
| `wiki_move_node` | 必须使用 `source_node_token` / `wiki_node_token`；不得使用 `wiki_obj_token`。 |
| `none` | 不执行移动，保留跳过原因。 |

### 计划 UI

```text
移动计划已生成：
- 默认将移动高相关：N 项
- 你已选择中相关：N 项
- 其中不可自动恢复：N 项
- 不会移动：N 项
- 无移动权限：N 项
- 移动权限未知：N 项

你可以回复“确认执行”，也可以继续调整分组、增减中相关资源，或取消本次移动。
```

## MovePlanItem

```json
{
  "plan_id": "稳定计划项 ID",
  "action_type": "create_target|move_resource|skip_resource|unsupported",
  "title": "资源或目标名称",
  "source_token": "按 move_method 选择的源 token",
  "source_node_token": "Wiki 节点移动使用的 node token",
  "target_token": "目标 token",
  "target_type": "drive_folder|wiki_node|wiki_space",
  "move_method": "drive_move|wiki_move_node|wiki_move_docs_to_wiki|none",
  "reason": "纳入或跳过原因",
  "rollback_supported": "是否支持自动恢复",
  "rollback_blocker": "不可自动恢复原因",
  "execution_status": "pending|success|failed|skipped"
}
```

| 字段 | 说明 |
|-------|------|
| `plan_id` | 稳定计划项 ID，用于连接计划、快照和执行日志。 |
| `action_type` | 计划动作类型。 |
| `source_token` | 源资源 token；具体含义由 `move_method` 决定。 |
| `source_node_token` | Wiki 节点移动时使用的 node token。 |
| `target_token` | 目标位置 token。 |
| `move_method` | 实际使用的移动方式。 |
| `rollback_supported` | 是否支持自动恢复。 |
| `rollback_blocker` | 不可自动恢复原因；Drive 文档移动到 Wiki 时默认为 `drive_to_wiki_not_reverse_movable`，表示回滚时不回迁、不删除迁入文档，只清理本次新建目标外壳。 |
| `execution_status` | 执行状态。 |

## ResourceItem

```json
{
  "title": "资源标题",
  "resource_type": "doc|docx|sheet|bitable|file|folder|wiki|slides|shortcut",
  "url": "资源链接",
  "canonical_token": "标准资源 token",
  "wiki_node_token": "Wiki 节点 token",
  "wiki_obj_token": "Wiki 底层对象 token",
  "wiki_obj_type": "Wiki 底层对象类型",
  "space_id": "知识空间 ID",
  "current_parent": "当前父级位置",
  "owner_id": "资源 owner open_id",
  "is_owner": "true|false|unknown",
  "permission_state": "readable|denied|unknown",
  "move_permission_state": "movable|denied|unknown",
  "move_permission_basis": "owner|explicit_full_access|api_denied|not_owner_unverified|metadata_unavailable|unsupported_direction|target_denied",
  "target_write_state": "confirmed|unknown|denied",
  "resolve_status": "resolved|partial|failed",
  "content_verify_state": "verified|search_evidence_only|skipped_by_move_permission|permission_denied|unverifiable|unsupported",
  "content_evidence": ["证据"],
  "relevance": "high|medium|low|permission_denied|no_move_permission|move_permission_unknown|unverifiable|unsupported_move_target"
}
```

| 字段 | 说明 |
|-------|------|
| `canonical_token` | 内容读取、Drive 对象操作或底层对象操作使用的标准 token；Wiki 节点移动不得使用该字段。 |
| `wiki_node_token` | Wiki 节点身份，用于 Wiki 节点移动。 |
| `wiki_obj_token` | Wiki 节点背后的真实文档 token。 |
| `current_parent` | 执行前父级位置，用于展示和恢复。 |
| `owner_id` | 资源 owner；Drive 资源优先来自 `drive metas batch_query`，Wiki 节点优先来自 `wiki +node-get`。 |
| `is_owner` | 当前用户是否为资源 owner。 |
| `permission_state` | 当前身份下的读取权限状态。 |
| `move_permission_state` | 当前身份下的移动资格状态；只有 `movable` 可进入默认移动链路。 |
| `move_permission_basis` | 移动资格判断依据，用于解释为什么纳入或排除。 |
| `target_write_state` | 目标位置是否确认可写。 |
| `content_verify_state` | 内容验证状态或跳过验证原因。 |
| `content_evidence` | 支撑相关性判断的命中证据。 |
| `relevance` | 相关性和可执行性分组。 |
