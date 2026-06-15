# 主题资料收集工作流

本文档是 `workspace-topic-move-collector` 的唯一入口，负责定义全局约束、状态机和渐进加载关系。具体阶段规则放在配套文档中，只有进入对应状态时才加载。

配套文档只是本 workflow 的引用文件，不是独立 skill。不要把用户请求直接路由到某个配套文档。

## 必读上下文

执行本 workflow 前，必须先阅读 [`../../lark-shared/SKILL.md`](../../lark-shared/SKILL.md)，用于处理身份、认证、权限和写操作确认规则。

按阶段渐进加载其他 skill / 引用文档：

- 目标是 Wiki 或个人文档库：[`../../lark-wiki/SKILL.md`](../../lark-wiki/SKILL.md)
- 需要读取文档内容：[`../../lark-doc/SKILL.md`](../../lark-doc/SKILL.md) 和 [`../../lark-doc/references/lark-doc-fetch.md`](../../lark-doc/references/lark-doc-fetch.md)
- 需要验证 Sheet 内容：[`../../lark-sheets/SKILL.md`](../../lark-sheets/SKILL.md)
- 需要 Drive 搜索：[`lark-drive-search.md`](lark-drive-search.md)
- 需要资源解析：[`lark-drive-inspect.md`](lark-drive-inspect.md)

## 适用范围

本 workflow 用于根据用户给出的主题、关键词或内容线索，在云空间 / 云盘 / Wiki / 电子表格等 Workspace 资源中查找相关资料，并在用户确认后统一移动到指定 Drive 文件夹或 Wiki 节点下。

适用触发语包括：

- "帮我找到和某主题相关的文档并放到这个文件夹"
- "把所有关于某项目的资料收集到知识库节点下"
- "找出包含某内容的资料，确认后移动到新建目录"
- "按这个关键词全量搜索云盘，把相关资料归档"

默认搜索范围是 Workspace 全量召回，不要求用户先限定文件夹或知识库范围。只有用户明确指定范围时，才使用 `--folder-tokens`、`--space-ids` 或其他显式限制。

## 非目标

默认不生成：

- 长篇研究报告
- 内容总结文档
- Sheet 清单或统计看板
- 自动权限治理报告

默认禁止执行：

- 未确认前创建文件夹或 Wiki 节点
- 未确认前移动资源
- 删除资源、重命名资源或修改公开权限
- 自动批量申请权限
- 把无权限或无法验证的资源加入移动计划
- 把移动权限未知或不具备移动资格的资源加入移动计划

如果用户明确要求把结果写入 Sheet / Doc，切到对应专项能力；本 workflow 的默认产物是移动后的资源归档结果。

## Agent 执行约束

触发本 workflow 后，agent 必须：

1. 按“执行状态机”的顺序执行。
2. 维护“运行时状态”中的字段。
3. 执行某个状态前，先读取本文档 `## 渐进加载关系` 表格中该状态对应的文档。
4. 用户可见说明、字段说明和 UI 文案使用中文。
5. 状态名、字段名、枚举值、命令名保留英文稳定标识。
6. 将 `CONFIRM_CONTEXT` 和 `CONFIRM_EXECUTION` 作为强用户确认门：前者确认主题、目标位置、身份、可选限制和目标解析结果后才能搜索；后者确认创建目标和移动资源后才能写入。
7. 进入 `EXECUTE` 前，不得创建目标文件夹 / 节点，也不得移动资源。
8. 必须展示每个相关性分组中的资源名称；低置信分组可以折叠，但必须可查看。
9. 默认只移动 `high` 相关资源；`medium` 资源必须由用户显式选择。
10. 即使用户可见列表分页展示，也必须维护完整内部状态。
11. `RESOURCE_RESOLVE` 和 `CONTENT_VERIFY` 是强制阶段，不得用搜索结果、标题或摘要直接替代。
12. 触发后锁定 `active_workflow=workspace-topic-move-collector`；执行期间不得自动切换到其他 workflow。
13. 如果认为需要切换 workflow，必须停止并向用户说明原因，等待用户确认。
14. `RESOURCE_RESOLVE` 是移动资格门禁；只有确认 `move_permission_state=movable` 的资源才能进入默认移动链路。

## 用户展示 UI 规则

所有用户可见 UI 都必须包含：

1. 已经完成的关键结果。
2. 下一步会做什么，以及是否会产生写操作。
3. 如果 `wait_for_user=true`，明确告诉用户可以选择的动作。
4. 如果无需用户操作，明确说明将继续执行，避免用户误以为流程停住。

典型动作包括：确认继续、修改主题 / 目标 / 限制、展开更多结果、调整相关性分组、选择中相关资源、确认执行、取消执行。

## 职责边界

| 文件 | 负责 | 不负责 |
|------|------|--------------|
| `lark-drive-workflow-topic-move-collector.md` | 触发规则、全局约束、状态机、渐进加载关系、命令族白名单 | 具体阶段规则、UI 模板、执行细节 |
| `lark-drive-workflow-topic-move-collector-setup.md` | `PARSE_INPUT`、`RESOLVE_TARGET`、`CONFIRM_CONTEXT`、`TargetLocation` | 搜索执行、相关性分类、写操作 |
| `lark-drive-workflow-topic-move-collector-recall.md` | `SEARCH_RECALL`、`RECALL_ENHANCE`、搜索 query 策略、去重、`CandidateItem` | 资源 token 解析、内容验证、写操作 |
| `lark-drive-workflow-topic-move-collector-review-plan.md` | `RESOURCE_RESOLVE`、`CONTENT_VERIFY`、`RELEVANCE_CLASSIFY`、`PLAN_MOVE`、`ResourceItem`、`MovePlanItem`、展示分组 | 写操作执行、恢复 |
| `lark-drive-workflow-topic-move-collector-execute.md` | `CONFIRM_EXECUTION`、`EXECUTE`、`VERIFY`、`RESTORE`、`RollbackSnapshotItem`、执行日志 | 搜索、分类和计划 schema |

## 运行时状态

agent 在一次 workflow 运行中必须维护以下内部字段：

| 字段 | 说明 |
|-------|------|
| `active_workflow` | 固定为 `workspace-topic-move-collector`，执行期间不得改写。 |
| `current_state` | 当前状态机节点。 |
| `topic` | 用户确认后的主题、关键词、同义词和排除词。 |
| `target_location` | 目标位置解析结果，见 setup 文件的 `TargetLocation`。 |
| `identity` | 执行身份；默认优先 `--as user`。 |
| `constraints` | 用户显式确认的类型、时间、创建人、范围等限制。 |
| `allow_cross_container_move` | 是否允许跨 Drive / Wiki 容器移动；默认允许，但必须展示给用户确认。 |
| `candidate_items` | 搜索召回结果，包含 query 证据和去重信息。 |
| `resource_items` | 解析后的标准资源列表。 |
| `relevance_groups` | 高相关、中相关、低相关、无权限、无移动权限、移动权限未知、无法验证、不可移动分组。 |
| `move_plan_items` | 经用户选择后生成的完整移动计划。 |
| `execution_journal` | 写操作日志，用于验证和恢复。 |
| `rollback_snapshot` | 写操作前位置快照，仅用于失败恢复或用户要求恢复。 |
| `display_page_state` | 用户可见列表的分页、筛选和展开状态。 |

## 执行状态机

| 状态 | 进入条件 | agent 必须执行 | 用户可见输出 | `wait_for_user` | 下一状态 |
|-------|-----------------|---------------|--------------------|---------------|------------|
| `PARSE_INPUT` | workflow 被触发 | 加载 setup 文档；解析主题、目标、身份和限制 | 澄清问题或解析摘要 | 必填字段缺失时为 `true` | `RESOLVE_TARGET` |
| `RESOLVE_TARGET` | 主题和目标已获得 | 解析已有目标，或解析待创建目标 | 目标解析结果 | 目标有歧义时为 `true` | `CONFIRM_CONTEXT` |
| `CONFIRM_CONTEXT` | 目标解析完成 | 展示主题、目标、身份、限制和跨容器设置 | 搜索前确认 UI | `true` | `SEARCH_RECALL` |
| `SEARCH_RECALL` | 用户确认上下文 | 用原始关键词和显式限制执行基础全量召回 | 搜索进度 / 基础统计 | 阻塞时为 `true` | `RECALL_ENHANCE` |
| `RECALL_ENHANCE` | 基础召回完成 | 执行覆盖增强 query 并合并结果 | 增强召回摘要 | 阻塞时为 `true` | `RESOURCE_RESOLVE` |
| `RESOURCE_RESOLVE` | 候选列表已准备 | 解析 token、类型、父级位置、owner 和移动资格 | 解析进度 / 阻塞摘要 | 阻塞时为 `true` | `CONTENT_VERIFY` |
| `CONTENT_VERIFY` | 资源列表已准备 | 对支持的资源做有界内容读取 | 验证摘要 | 阻塞时为 `true` | `RELEVANCE_CLASSIFY` |
| `RELEVANCE_CLASSIFY` | 证据已准备 | 按相关性和可执行性分组 | 分组结果列表 | `false` | `PLAN_MOVE` |
| `PLAN_MOVE` | 分组完成 | 基于默认规则和用户可选项生成移动计划 | 草案计划和选择项 | `true` | `CONFIRM_EXECUTION` |
| `CONFIRM_EXECUTION` | 用户要求执行 | 展示创建、移动、跳过项和风险 | 写操作确认 UI | `true` | `EXECUTE` 或 `PLAN_MOVE` 或 `DONE` |
| `EXECUTE` | 用户明确确认写操作 | 需要时先创建目标，再移动确认资源 | 执行进度 | 阻塞时为 `true` | `VERIFY` 或 `RESTORE` |
| `VERIFY` | 执行完成 | 验证目标位置下的移动结果 | 验证结果 | 提供恢复选项时为 `true` | `DONE` 或 `RESTORE` |
| `RESTORE` | 用户要求恢复 | 仅基于快照和日志恢复 | 恢复确认 / 结果 | 写操作前为 `true` | `VERIFY` 或 `DONE` |
| `DONE` | 无后续操作 | 停止 | 最终回复 | `false` | 结束 |

### 状态跳转硬约束

1. `RECALL_ENHANCE` 完成后，下一状态必须是 `RESOURCE_RESOLVE`；不得直接进入 `RELEVANCE_CLASSIFY` 或 `PLAN_MOVE`。
2. `RESOURCE_RESOLVE` 必须为每个 `CandidateItem` 生成对应的 `ResourceItem`，或生成明确的解析失败 / 权限受限状态。
3. `RESOURCE_RESOLVE` 必须为每个 `ResourceItem` 写入 `move_permission_state` 和 `move_permission_basis`。
4. `CONTENT_VERIFY` 必须为每个 `ResourceItem` 写入内容证据、搜索证据复用说明，或不可验证原因；移动权限未知或无移动权限的资源可以只写入跳过验证原因。
5. 只有当 `resource_items` 已准备且每项都有验证状态或跳过验证原因时，才能进入 `RELEVANCE_CLASSIFY`。
6. 用户调整相关性分组后，必须回到 `RELEVANCE_CLASSIFY` 输出调整后的分组结果，再进入 `PLAN_MOVE` 重新生成计划。

### Workflow 切换门禁

只有以下情况允许考虑切换 workflow：

1. 用户明确说不再做主题资料收集，改为整理整个目录结构或生成盘点方案。
2. 当前 workflow 明确无法覆盖用户的新目标。
3. 用户要求的是目录结构治理，而不是查找主题相关资料并移动。

即使满足以上条件，也不得自动切换；必须先向用户说明原因并等待确认。

## 渐进加载关系

| 状态 | 必读文档 |
|-------|---------------|
| `PARSE_INPUT` / `RESOLVE_TARGET` / `CONFIRM_CONTEXT` | [`lark-drive-workflow-topic-move-collector-setup.md`](lark-drive-workflow-topic-move-collector-setup.md) |
| `SEARCH_RECALL` / `RECALL_ENHANCE` | [`lark-drive-workflow-topic-move-collector-recall.md`](lark-drive-workflow-topic-move-collector-recall.md) |
| `RESOURCE_RESOLVE` / `CONTENT_VERIFY` / `RELEVANCE_CLASSIFY` / `PLAN_MOVE` | [`lark-drive-workflow-topic-move-collector-review-plan.md`](lark-drive-workflow-topic-move-collector-review-plan.md) |
| `CONFIRM_EXECUTION` / `EXECUTE` / `VERIFY` / `RESTORE` | [`lark-drive-workflow-topic-move-collector-execute.md`](lark-drive-workflow-topic-move-collector-execute.md) |

## 命令映射

| 状态 | 允许的命令族 | 用途 |
|-------|--------------------------|---------|
| `RESOLVE_TARGET` | `drive +inspect`、`wiki +node-get`、`wiki +space-list`、仅用于查找文件夹候选的 `drive +search` | 解析目标位置 |
| `SEARCH_RECALL` / `RECALL_ENHANCE` | `drive +search` | 搜索召回和覆盖增强 |
| `RESOURCE_RESOLVE` | `drive +inspect`、`wiki +node-get`、`drive metas batch_query`、必要时 `drive permission.members auth` | 解析标准 token、owner、权限信号和移动资格 |
| `CONTENT_VERIFY` | `docs +fetch`、`sheets +read`、`sheets +find`、必要时 `drive +preview` | 验证内容证据 |
| `EXECUTE` | `drive +create-folder`、`wiki +node-create`、`drive +move`、`wiki +move`、`drive +task_result` | 执行已确认写操作 |
| `VERIFY` | `drive files list`、`wiki +node-list`、`wiki +node-get`、`drive +inspect`、`drive +task_result` | 验证执行结果 |
| `RESTORE` | `drive +move`、`wiki +move`、`drive +delete`、`wiki +node-delete`、`drive +task_result` | 恢复已确认资源并清理本次新建目标 |

## 引用文档

- [输入与目标确认](lark-drive-workflow-topic-move-collector-setup.md)
- [召回](lark-drive-workflow-topic-move-collector-recall.md)
- [审核与计划](lark-drive-workflow-topic-move-collector-review-plan.md)
- [执行](lark-drive-workflow-topic-move-collector-execute.md)
- [lark-drive-search](lark-drive-search.md)
- [lark-drive-inspect](lark-drive-inspect.md)
- [lark-drive-move](lark-drive-move.md)
- [lark-drive-create-folder](lark-drive-create-folder.md)
- [lark-drive-delete](lark-drive-delete.md)
- [lark-wiki-move](../../lark-wiki/references/lark-wiki-move.md)
- [lark-wiki-node-create](../../lark-wiki/references/lark-wiki-node-create.md)
- [lark-wiki-node-delete](../../lark-wiki/references/lark-wiki-node-delete.md)
