# Dida Kanbanloop Skill Implementation Plan

> **For agentic workers:** Use the installed `swarm-cluster` skill for approved parallel work, or follow the current environment's in-session plan execution workflow. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 交付一个自包含 OpenCode Skill，使单个 Dida365/TickTick 父任务沿 `处理中` → `待审查` 完成真实工作与子任务同步，并把 `已完成` 保留给用户审查。

**Architecture:** 根 `SKILL.md` 承载唯一运行时工作流，以只读预检、受限 MCP 写入、写后证据和幂等重入构成状态约束；README 负责安装、精确列配置与人工审查指引。构建只增加 Markdown，验证采用 Python 3 标准库内联静态检查和 14 个无真实写入的场景账本走查。

**Tech Stack:** OpenCode Skill Markdown、Dida365/TickTick MCP、Markdown README、Python 3 标准库（仅构建期检查）；无运行时依赖。

**Spec:** `docs/superpowers/specs/2026-10-27-dida-kanbanloop-skill-design.md`（执行者必须同时阅读 UR-1～UR-5、DC-1～DC-5、AD-1～AD-9 与 AC-01～AC-13）

## Global Constraints

- 所有实施与验证命令均从仓库根目录运行；路径均相对该根目录解析。Build 前记录完整 status 基线，Build 后比较，既有未跟踪路径单独保留。
- 每次调用只处理一个由“清单/项目 + Kanban 看板 + 父任务”唯一确定的目标；缺信息或匹配歧义时零写入停止/询问。标题无 active 匹配时必须要求 ID 区分未找到/已结束；`parentId` 非空的目标不是父任务，零写入。
- 三个工作流列 `处理中`、`待审查`、`已完成` 各精确存在一次；空白/近似名不能替代。其他列允许存在，但必须忽略且永不修改，不得自动创建、改名或重排任何列。
- 所有目标、权限、列和父子任务完整性检查都在首次写入前通过；父任务必须在顶层且 `status = 0`。
- 父任务只允许通过 `update_task(task_id, task: { columnId })` 进入 `处理中` 或 `待审查`；禁止完成父任务或自动移到 `已完成`。
- 子任务只指 `parentId = parent.id` 的 child task；必须以 `filter_tasks({ filter: { projectIds, status: [0, -1] } })` 枚举 active + abandoned，以 completed 日期查询补齐 `status = 2`，并逐读父与每个子任务的 `title/content/desc`。任一分支不可用、截断或范围不全即零写入，不能用 undone-only 结果代替。
- 父任务下所有 active 子任务默认属于本轮范围；只有用户明确排除或原先已 abandoned 的项可不计入待完成集合，Skill 不得自行排除或放弃；最终报告列出这些 ID/status/原因。
- 合适活跃子任务必须复用；已完成且仍满足当前要求的子任务保留计完成，不创建重复 Agent 项。新建标题采用 `N. <动作> [Agent]`，创建仅传 `projectId`、`parentId`、`title` 和可选初始 `content`，不传 `columnId/status/sortOrder` 或其他字段。
- 子任务创建后禁止 `update_task`、`move_task`、批量完成、重开、删除或任何列/标题/描述/顺序修改；仅实际完成后可单个 `complete_task`，完成前重读 ID/项目/父关系/status。
- 每次完成子任务都要有真实工作证据，并核实 `status = 2`、`parentId` 不变、`columnId` 不变；不得通过反向移动修复异常。失去必要性的 child 保持 incomplete 并报告。
- 正常本地项目/文档进度按目标项目既有规则和用户习惯继续，不强制新增进度文件。
- 每个 Dida 成功声明必须有成功 MCP 响应的目标字段或写后回读；创建结果不明时不得盲目重试；部分成功按 confirmed `taskId + status` 与 pending actions 报告。
- 父任务到 `待审查` 并核实仍 active 后立即停止；用户通过移到 `已完成`，拒绝则更新描述并移回 `处理中` 后再调用。
- **Spec/Plan 修订只允许发生在 pre-Build review loop。Build 只能创建根 `SKILL.md`、修改 `README.md`，以及对本 Plan 的 Audit Record 做簿记；Build 不得修改 Spec 或 Plan 的其他部分。** 不增加运行时脚本、依赖、安装器、测试夹具或无关文件，不修改 `LICENSE`。
- 未经当前会话明确授权不得 stage、commit、push；本计划不包含提交步骤。

## Review Focus

1. **目标或历史不完整**：同名/child 目标、额外或近似列、`filter_tasks` 截断、undone-only 或 completed 日期/上限可能让代理误选或漏读；合理行为是首次写入前停止，只有三个必需列均精确存在时才允许忽略其他额外列。由 Task 3 Step 2 的 S1～S4、S12～S14 固定。
2. **子任务被过度修改**：去重、排序或异常恢复可能诱导改标题、重排、批量完成或回移列；合理行为是仅补建和有证据的单个子任务完成。由 S6、S7 固定。
3. **人工审查门被绕过**：已完成工作可能诱导完成父任务、自动进入 `已完成` 或从 `待审查` 自动拉回；合理行为是父保持 active 并停止。由 S8、S12 固定。
4. **不确定写入与重入造成重复**：创建超时、部分成功、拒绝后返工可能产生重复子任务或虚假成功；合理行为是先回读/reconcile、限制重试并保留账本。由 S9～S11 固定。
5. **文档验证被误当实时验证**：静态关键词和本地工作证据不能证明真实 MCP 状态；合理行为是区分检查层级，没有授权夹具就明确未做实时端到端验证。由 S5～S9 和 Task 3 Step 4/5 固定。

---

## File Map

- **Create: `SKILL.md`** — 唯一 Skill 定义；front matter、目标解析、只读预检、父子任务工作流、MCP 白名单、证据、失败/重试和重入。
- **Modify: `README.md:1-16`** — 全部替换旧 `待办` 自动同步说明；提供安装、配置、调用、审查和故障指引。
- **Read only during Build: `docs/superpowers/specs/2026-10-27-dida-kanbanloop-skill-design.md`** — 权威需求、决策与验收来源；只有 pre-Build review loop 可修订，Build 不得写入。
- **Audit bookkeeping only during Build: `docs/superpowers/plans/2026-10-27-dida-kanbanloop-skill.md`** — 只有 pre-Build review loop 可修订计划正文；Build 仅能记录审查、基线和验证结果，不得改任务/接口/验收。
- **Verify only: `LICENSE` 及 status 基线中的既有未跟踪路径** — 不修改、不清理、不回退。
- **Test files:** 不创建；所有静态检查使用内联 Python，场景走查不调用真实 MCP。

## Audit Record

- review_round:3/3
- W4 independent pre-Build review verdict: REVISE — 0 BLOCKER / 7 MAJOR / 6 MINOR.
- 当前阶段：W5 round 2 独立 pre-Build 审查已于 2026-10-27 PASS；Build 已获授权，但仅限 Plan 声明的范围。W3 本次仅簿记审查结果，未更改 Spec/Plan 需求。
- `independent_pre_build_review`：**PASS**（reviewer W5，2026-10-27，`review_round:2/3`）；Severity counts：0 BLOCKER / 0 MAJOR / 0 MINOR。此前 M1–M7 与 m1–m6 均已解决，无新增阻塞问题。
- `W5_review_evidence`：审阅 Spec 271 行、Plan 365 行；核实 `filter_tasks({ filter: { projectIds, status: [0,-1] } })` abandoned 路径、completed/abandoned 父任务 ID 规则、精确三列且额外列忽略语义、Build 范围约束、含 S5/S7 的 AC-11 追踪、S14 child-as-parent 零写入、S6 completed+satisfied 复用；确认 AC-01–AC-13 均有映射、S1–S14 均有表达，且未声称实时 MCP 验证。W5 另确认 repo status 仅有两个未跟踪审计产物、`SKILL.md` 缺席、README 16 行、LICENSE 存在。
- **编辑权限**：Spec/Plan 修订只在 pre-Build review loop 进行；Build 不得修改 Spec 或 Plan 正文/任务/接口/验收，只能创建 `SKILL.md`、修改 `README.md` 并在此处做 Audit Record 簿记。
- **W4 revisions applied（2026-10-27）**：
  - M1：Spec §5.1.6/§6/AC-04/AC-10 与 Plan S4/S13/静态列表加入 `filter_tasks(projectIds,status:[0,-1])` 权威 active+abandoned 分支；工具不可用、capped/truncated 或完整性不明均 `W0`，不以 undone 快照替代。
  - M2：Spec AD-8/§5.1.5 与 Plan S12 规定标题无 active 匹配时必须要求显式任务 ID；分别报告未找到、已完成和已放弃，始终零写入。
  - M3：Spec AD-2/§8/AC-03 与 Plan S3 统一为“三个必需列各精确一次，额外列允许但忽略且不修改”；增加额外列、尾部空白和近似名变体。
  - M4：Plan Global Constraints/File Map/Task 1/Task 3/Execution Handoff/Audit Record 锁定编辑权限；Spec/Plan 只能在 pre-Build review loop 修订，Build 仅能写 `SKILL.md`、`README.md` 和 Plan Audit Record。
  - M5：Spec AC-11 与 Plan AC-11/S9 纳入首次父移动 S5、child 完成 S7，并要求部分成功报告 confirmed `taskId + status` 和 pending actions。
  - M6：新增 S14，覆盖 `parentId` 非空的 child 被指定为父任务时零写入并要求顶层父任务；映射 AC-02/AC-07。
  - M7：扩展 S6，已完成且当前证据仍满足要求的 child 直接复用计完成，不创建重复 Agent child；映射 AC-06。
  - m1：Task 1/Task 3 静态检查加入字面模式 `N. <动作> [Agent]` 与 `status = -1`。
  - m2：README 静态检查改为覆盖 Spec §8 的八组内容，并以无引号限制的 `待办` 缺席检查识别旧流程。
  - m3：S4/S6/S7/S8/S9 增加空描述、用户习惯顺序、失去必要性、完成前身份复核、限流退避、abandoned/用户排除项最终报告及 `create_task` 字段白名单变体。
  - m4：Spec §5.1.6/§6 与 Plan S13 定义 completed 初始日期策略：可靠父创建时间优先，否则项目限定无日期初查；需要日期或 capped 时取得覆盖父生命周期的用户范围并无间隔分窗，仍截断即 `W0`。
  - m5：Task 1 记录 pre-Build `git status --short --untracked-files=all` 基线；Task 3 比较 after/before，新增路径仅允许 `SKILL.md`/`README.md`，既有未跟踪路径独立保留。
  - m6：强化 placeholder 正则和 front matter 起止解析；post-Build handoff 纳入 Plan/Audit Record；追踪矩阵定义 `Tn-Stepk` 与 `Sn` 记法。
- `pre_build_status` / `post_build_status`：由 Task 1/Task 3 记录真实输出和路径集合比较，既有未跟踪路径不得被清理。
- `build_static_checks`：由 Task 3 Step 1 的实际输出填写，禁止预写 PASS。
- `scenario_walkthrough`：由 Task 3 Step 2 按 `通过数/14` 和修订记录填写。
- `independent_post_build_review`：由 Task 3 Step 5 的全新只读上下文给出 verdict、问题计数和证据；不能用构建者自查替代。
- 同一 `(Plan/Spec 路径, 2026-10-27)` 目标线如 round 2 仍需修复并再次独立审查，则更新为 `review_round:3/3`；第四轮必须另获用户明确授权。每轮只修上一轮审计发现，不夹带新范围。
- **W6 Build record（2026-10-27）**：
  - `pre_build_status`（Build 写入前原文）：
    ```
    ?? docs/superpowers/plans/2026-10-27-dida-kanbanloop-skill.md
    ?? docs/superpowers/specs/2026-10-27-dida-kanbanloop-skill-design.md
    ```
    基线路径集合为上述两个计划/规格文档；`test ! -e SKILL.md` 退出 0。
  - `pre_build_hashes`：README `95a1fcc190e37eb440b98ff181400d43d4996076c1c7227c5d185b055cd39f20`、LICENSE `959b2aa569965d40f07a99c21c2ff15ca1a7ed499d1cda78c2b01471cb0daf71`、Spec `e2bda519b10991bbc4a5d7420fb24170ffe10d8ef08820bc8580afe8507b2fb6`、Plan `659bde921ec0c4e3f7d3e8299163d447c6a624f4daa5336fd570918cb26ad621`。
  - `build_static_checks`：Task 1 `SKILL.md focused static checks: PASS`；Task 2 Step 1 对旧 README 为预期 FAIL（`AssertionError`，exit 1）；Task 2 Step 3 `README.md focused static checks: PASS (8/8 onboarding groups)`；Task 3 Step 1 `Combined static acceptance: PASS (2 files; README 8/8 groups)`。
  - `scenario_walkthrough`：S1–S14 逐条走查 **14/14 PASS**；仅文档契约与变更账本推演，未调用真实 MCP。修订：S11 走查时在 SKILL §10 补明“审查通过、拒绝、更新描述和移回 `处理中` 均为用户动作，不记作 Skill 写入”；S4 走查时补充“用户习惯或依赖要求先做父级探测/集成时可调整顺序”；两处措辞补充后均重跑 Task 3 Step 1，仍 PASS。
  - `post_build_status`（Task 3 Step 3 原文）：
    ```
     M README.md
    ?? SKILL.md
    ?? docs/superpowers/plans/2026-10-27-dida-kanbanloop-skill.md
    ?? docs/superpowers/specs/2026-10-27-dida-kanbanloop-skill-design.md
    ```
  - `build_scope_check`：基线未跟踪路径原样保留；`after_untracked - before_untracked = {SKILL.md}`；tracked 实施改动仅 `README.md`；Spec sha256 前后一致 `e2bda519…`；LICENSE 前后一致 `959b2aa5…`；Plan 与审计前副本的 diff 仅包含本 Audit Record 块；未 stage/commit/push，未改 LICENSE。
  - `validation_layer_record`：Task 1/2/3 静态检查 PASS；S1–S14 场景走查 14/14 PASS；**未做实时 MCP 端到端验证**（无专用测试清单/看板/父任务夹具，本轮禁止真实 Dida 调用）。静态与场景 PASS 不等于运行时 PASS。
  - `independent_post_build_review`：**PASS**（reviewer W7，2026-10-27，全新只读上下文，`review_round:2/3`）；Severity counts：**0 BLOCKER / 0 MAJOR / 7 MINOR**。W7 完成两遍审查：pass 1 查缺陷/遗漏/回归，pass 2 追溯原始需求 UR-1～UR-5 与 AC-01～AC-13。全部原始需求在文档/契约层均满足，AC-01～AC-13 全部覆盖；未声称也未执行实时 MCP 端到端验证，W6 未自审、未预写 verdict。
- **W7 post-Build review record（2026-10-27）**：
  - `W7_review_evidence`：W7 仅只读审查，未调用 Dida MCP，未做 Git 变更。其复核的静态结果：`SKILL.md focused static checks: PASS`；`README.md focused static checks: PASS (8/8 onboarding groups)`；`Combined static acceptance: PASS (2 files)`；`git diff --check` 无输出；status 恰为 `M README.md`、`?? SKILL.md` 及两份审计文档；pre-Build 哈希与范围检查一致。
  - `W7_minor_findings`（7 项，均 MINOR；不阻塞交付、不允许任何不安全写入、本次交付不修复，Skill 随以下披露交付）：
    1. Plan 的 Execution Handoff 仍要求分派 round 2 pre-Build 复审，但 Build 与 W7 均已完成（过时的 handoff 措辞）。
    2. README 未明确写出“重复的 active 父任务标题必须提供任务 ID”，而 `SKILL.md` 有此要求。
    3. `SKILL.md` §3.5 的 status-first 报告方式，在按 ID 选中已完成/已放弃 child 时可能弱化“child 不得作为父任务”的拒绝信息；零写入结论仍成立。
    4. 最终父任务回读未显式把 `columnId` 再与审查/完成列核对一次，以覆盖少见的运行中外部移动。
    5. 工具表含 `client_timezone`，但 completed 分支工作流未再显式重述。
    6. Audit Record 的场景证据为摘要级（W7 已独立重走全部 14 个场景），且一处旧行数引用差一。
    7. README 称列顺序为 “strict”，而 `SKILL.md` 按精确 ID 解析且不检查 `sortOrder`。
  - `review_round` 说明：保持 `2/3`；W7 的 post-Build 审查为 PASS，无需再开下一轮独立复审。
- **W11 follow-up（2026-10-27）**：应用两项 W7 MINOR（#4/#7）、G2 用户澄清，并在 W14 对齐 Spec 措辞。W11 阶段仅改 `README.md`、`SKILL.md` 以及本 Audit Record 簿记，不改 Spec、任务、接口、AC 矩阵、S1–S14 或需求；当时 `review_round:2/3` 不变。W14 的文件范围与轮次见下方 W14 记录。
  - G1（对应 W7 MINOR #7）已解决：README 列配置改为“名称必须精确为（建议显示顺序）”，保留 `处理中` → `待审查` → `已完成` 展示，并新增“按列 ID 解析状态、不依赖视觉顺序”。不新增 `sortOrder` 检查：视觉顺序不是用户需求，强制校验只会造成假停止。
  - G2 已获用户确认：README 与 `SKILL.md` §3.3 均澄清“看板”指目标清单的同一 Kanban 视图，三个必需名称是该视图中的列名，不是三个独立看板；运行时行为不变。
  - G3（对应 W7 MINOR #4）已解决：`SKILL.md` §7 最终确认改为同时要求返回/回读显示 `columnId == reviewColumn.id` 与 `status = 0`；§11 结束前自问同步加入该列 ID 条件；未新增工具或写入行为。
  - `W11_static_checks`（实际输出，2026-10-27）：`SKILL.md focused static checks: PASS`；`README.md focused static checks: PASS (8/8 onboarding groups)`；`Combined static acceptance: PASS (2 files; README 8/8 groups)`；`git diff --check` 无输出，exit 0。
  - `W11_diff_scope`：仅 `README.md`、`SKILL.md` 与本 Plan 的 Audit Record；`git status --short --untracked-files=all` 为 `M README.md`、`M SKILL.md`、`M docs/superpowers/plans/2026-10-27-dida-kanbanloop-skill.md`；未 stage、未 commit、未 push，未做实时 MCP 端到端测试。
- **W14 consistency pass（2026-10-27）**：
  - `scope`：Spec UR-5/§8.4/AC-03/AC-12 已对齐为“三个精确列名 + 建议显示顺序；运行时按列 ID 解析；视觉顺序不作运行时强制”。未改 AD-2、行为规则、任务、接口、S1–S14 或 AC 矩阵结构。
  - `reason`：仅消除 W12 指出的 Spec/README 措辞张力，不改变行为或用户要求；G2 继续保持“同一 Dida 清单的同一 Kanban 视图中的三个列”语义。
  - `review_round:3/3`：Spec/Plan 目标线已进行此次措辞修订，将接受一次独立 focused re-review；本记录不代表该复审已经给出 verdict。
  - `verification`（实际输出）：`SKILL.md focused static checks: PASS`；`README.md focused static checks: PASS (8/8 onboarding groups)`；`Combined static acceptance: PASS (2 files; README 8/8 groups)`；指定 `grep -n "名称及顺序\|名称和顺序\|精确名称及顺序"` 对 Spec/README/SKILL 无剩余命中（exit 1），无 G1 矛盾残留；未做实时 MCP 测试。

## Scenario Fixtures Used by Tasks

下表是文档行为的测试 oracle，不是实时 MCP 调用脚本。`W0` 表示零 Dida 写入且未开始目标项目工作；所有场景都必须同时检查“允许的调用顺序、禁止调用、最终状态、成功/失败报告”。

| ID | 输入/故障夹具 | 唯一可接受的预期结果 |
|---|---|---|
| **S1** | 分别缺少清单/项目、看板、父任务；另设同名项目、同名父任务和无法唯一绑定的看板 | 一次询问全部缺失/消歧项，`W0`；不得选首个模糊结果 |
| **S2** | MCP 不可用/未认证；项目 `permission = read` 或 `comment`；`viewMode` 明确非 Kanban | 报告具体配置/权限阻塞，`W0`；不模拟同步、不以写入探测权限 |
| **S3** | 分别设置：`处理中` 缺失、`待审查` 重名、列属于其他 `projectId`；三个精确列加额外 `归档` 列；以 `处理中 `/`已完成 ` 或 `待审核` 替代必需列 | 缺失/重名/错项目/空白/近似名替代均指出人工修正且 `W0`，禁止 `create_column`/`update_column`；三个必需列各精确一次时，额外列变体允许继续且 `归档` 永不修改。近似名只有不能替代必需列时才触发拒绝 |
| **S4** | 父为 CHECKLIST 用 `desc`；active child 分别使用 `content`/`desc` 或两者为空；另有 completed 和 abandoned child；含 `items`。`filter_tasks` 返回 active+abandoned；用户习惯变体要求先做父级探测再做 child | 必须以 `filter_tasks(projectIds,status:[0,-1])` 和 completed 分支枚举，再逐读 `title/content/desc/status/parentId/columnId`；空描述算已读取，`items` 不当 child，abandoned 不重开/完成。通常 child 优先，但用户习惯变体按依赖调整并说明依据；active child 默认属于范围 |
| **S5** | 完整预检通过，父在其他列且 active，有两个活跃 child | 第一次写入仅为父 `update_task(columnId = 处理中)`；响应/回读证明父 `taskId + columnId + status = 0` 后才规划/工作；其他任务启动时零移动。父已在 `处理中` 的变体不重复更新 |
| **S6** | 分别设置：A 为合适活跃 child；B 无对应项；C 为已完成且当前产物/验证仍满足要求的 child；D 在计划变化后失去必要性。项目已有本地进度规则 | A 复用不改名/重排；C 核对当前证据后保留 `status = 2` 并计完成，不创建重复 Agent 项。B 仅创建一次，示例 `2. 完成剩余实现 [Agent]`，只允许 `projectId/parentId/title/content?`，不得传 `columnId/status/sortOrder/dueDate/tags` 等字段；响应/回读确认 ID、项目、父关系、active、后缀。D 保持 incomplete/active，报告 ID/原因并阻止审查直至用户排除或处理；本地进度按既有规则更新 |
| **S7** | child 尚无工作证据，随后有真实测试证据；完成前复读到错误项目/父关系/status 的变体；完成响应分别完整、缺字段、显示 `columnId` 改变 | 无证据不完成；完成前必须以 `get_task_in_project` 复核 `taskId/projectId/parentId = parent.id/status = 0` 并记录列，任一不符零写入。有证据且匹配后仅 `complete_task(project_id, child_id)`，响应或补读证明 `taskId + status = 2`、父关系与列不变才报完成；列异常停止且不回移，父不进审查 |
| **S8** | 所有请求工作、验证、本地进度和范围内 child 均已有证据，父仍 active；另有一个原先 abandoned child 和一个用户明确排除的 active child | 仅把父 `columnId` 更新为 `待审查`，响应/回读仍为 `status = 0` 后停止；无父完成、自动 `已完成` 或后续创建。最终报告列出 abandoned 的 `taskId + status = -1` 及用户排除项的 `taskId + status = 0 + 排除原因`，不得由 Skill 改变它们 |
| **S9** | 只读瞬时错误、显式 rate limit/退避；初始父移动/子完成/最终父移动结果不明或失败；最终移动前已有部分成功 | 普通读最多一次合适重试；限流遵守服务端 backoff，无退避信息不紧密循环。写结果先回读，满足条件才最多重试一次；无证据不报成功。初始失败不开始工作；后续失败保留父于 `处理中`。所有部分成功报告逐项包含 confirmed `taskId + status`（父另含 columnId）和 pending actions；最终失败只报工作实际完成，不报审查交接完成 |
| **S10** | `create_task` 结果不明；回读可能找到一个匹配项或多个歧义项；随后以父在 `处理中` 的会话重入 | 先通过 `filter_tasks` 与 completed 分支枚举 child；唯一匹配则复用且不再次创建，歧义则停止，缺失时不立即盲试；重入重新预检，不重复父移动或已有步骤 |
| **S11** | 用户在 `待审查` 后更新父描述为“补充边界测试”并手动移回 `处理中`；旧 child 有完成项 | 用户动作不计为 Skill 写入；新调用读取最新父描述和全部 child/本地产物，复用仍有效项，不重开旧项，为修正创建有区分度的 Agent 步骤，实际完成后才再次进入 `待审查` |
| **S12** | 父分别在 `待审查`/`已完成`；标题只有已完成或 abandoned 候选而无 active 匹配；随后分别提供有效 ID、不存在 ID；另有外部完成父任务的并发变体 | 所有变体 `W0` 或立即停止相关写入。标题无 active 匹配时只报告“未找到 active 父任务”并要求 ID，不猜已结束；ID 读取到 `status = 2`/`-1` 才报告已完成/已放弃，不存在或项目不符报告未找到。不得自动拉回、重开、完成或补偿性移动 |
| **S13** | 分别设置：`filter_tasks` 不可用/错误/capped/truncated；undone 快照无 child 且 completed 无可靠父创建时间，先做 `projectIds` 限定无日期初查；初查被截断、服务要求日期、用户范围不能覆盖父生命周期或分窗仍 capped | 每个变体都报告具体分支/状态/日期缺口并 `W0`，连父 `处理中` 移动也不执行；不得以 undone 推断无 child。completed 优先用可靠父创建时间，否则先项目限定无日期初查；需要日期或 capped 时仅在用户确认完整范围下无间隔分窗，任一截断/缺口即停止，不得直接创建新计划 |
| **S14** | 用户指定的“父任务”经 ID 或 active 标题解析后具有非空 `parentId`，实际是另一任务的 child | 说明选中的是 child，要求提供顶层父任务，`W0`；不得把它移动到 `处理中`、完成它或把它当作新 child 的父任务 |

### Task 1: 构建自包含 Skill 工作流

**Files:**
- Create: `SKILL.md`
- Read: `docs/superpowers/specs/2026-10-27-dida-kanbanloop-skill-design.md`
- Test: 内联 Python；不创建测试文件

**Dependencies:**
- W3 的 Spec 与 Plan 均可读。
- Audit Record 中已有 round 2 全新上下文的 pre-Build PASS；若仍为修订项，停止并交回 Tower 的 pre-Build review loop。Build 不得修改 Spec/Plan 来消除审查发现。

**Interfaces:**
- Consumes: UR-1～UR-5、DC-1～DC-5、Spec §4～§7 的状态机、工具参数和 S1～S14 oracle。
- Produces: front matter `name: dida-kanbanloop-skill` 与触发 `description`；README 和审查者共同依赖的工作流章节、精确 MCP 名称、状态值、证据规则与禁止调用约定。

- [ ] **Step 1: 核对 pre-Build 审查门和创建范围**

检查 Audit Record 的 round 2 独立 verdict、Spec 路径以及根目录是否尚无 `SKILL.md`。在任何 Build 写入前记录基线：

```bash
git status --short --untracked-files=all
test ! -e SKILL.md
```

Expected: pre-Build 为 PASS 且两个命令退出 0；把 status 原文和既有路径集合写入 Audit Record 的 `pre_build_status`，既有未跟踪路径单独列为保留对象。若 `SKILL.md` 意外存在或审查未通过，停止并交回 Tower，不覆盖未知内容；不得为获得“干净”基线而清理文件。

- [ ] **Step 2: 创建 `SKILL.md` 元数据和不可协商规则**

写入 YAML front matter，固定 `name: dida-kanbanloop-skill`，`description` 必须说明何时用于单个 Dida365/TickTick 父任务及三列人工审查闭环。正文开头列出 Spec §4.2 的不变量，明确父/子任务允许与禁止的写入，不能只依赖后文隐含约束。

Expected: 文件无需任何运行时代码即可被 OpenCode 识别；名称与仓库一致，父任务永不完成、子任务永不移列在规则区直接可见。

- [ ] **Step 3: 写入唯一运行时流程和 MCP 契约**

在同一文件中依次编写：调用即使用 MCP、目标补齐与精确解析、项目/权限/看板/列预检、父及全部 child 完整读取、父进入 `处理中`、内容驱动排序、复用活跃及已完成且满足的 child、补建有序 child、真实工作与单项完成、父进入 `待审查` 后停止。加入 Spec §6 的参数表、明确写成 `status = 0`、`status = -1`、`status = 2` 的状态值、写后证据，以及 §7 的失败、限流退避、有限重试、并发和全部重入规则。

必须明确：标题无 active 匹配时要求 ID 区分未找到/已结束，child 不能作为父目标；`filter_tasks({ filter: { projectIds, status: [0, -1] } })` 是 active+abandoned 的必需枚举分支，不可用/截断即停止；`list_completed_tasks_by_date` 优先按可靠父创建时间起查；无可靠起点时先做 `projectIds` 限定、无日期收窄的初查，服务要求日期或初查 capped 时取得用户确认的完整生命周期范围并无间隔分窗，任一 capped/缺口即停止。新 child 标题为 `N. <动作> [Agent]`，`create_task` 只传 `projectId/parentId/title/content?`；本地进度沿用目标项目习惯；结果不明的创建先 reconciliation；不创建“等待审查”子任务。

Expected: S1～S14 每一步都能在该文件找到唯一行为，且首次 Dida 写入之前已经完成全部只读预检。

- [ ] **Step 4: 运行 Skill 聚焦静态检查**

Run:

```bash
python3 - <<'PY'
from pathlib import Path
import re
s = Path('SKILL.md').read_text(encoding='utf-8')
fm = re.match(r'\A---\n(.*?)\n---\n', s, re.S)
assert fm, 'SKILL.md must start with closed YAML front matter'
front = fm.group(1)
assert len(re.findall(r'(?m)^name: dida-kanbanloop-skill$', front)) == 1
assert re.search(r'(?m)^description:\s*\S.*$', front), 'description must be non-empty'
required = [
    '处理中', '待审查', '已完成', 'N. <动作> [Agent]', '[Agent]',
    'parentId', 'columnId', 'content', 'desc', 'list_projects',
    'get_project_with_undone_tasks', 'filter_tasks', 'list_columns',
    'get_task_in_project', 'list_completed_tasks_by_date',
    'update_task', 'create_task', 'complete_task',
    'status = 0', 'status = -1', 'status = 2',
]
missing = [token for token in required if token not in s]
assert not missing, f'missing Skill contract tokens: {missing}'
assert s.endswith('\n')
print('SKILL.md focused static checks: PASS')
PY
```

Expected: 输出 `SKILL.md focused static checks: PASS`；若失败，只修改 `SKILL.md` 补足实际契约后重跑同一检查。

### Task 2: 重写 README 安装与人工审查指引

**Files:**
- Modify: `README.md:1-16`
- Read: `SKILL.md`
- Test: 内联 Python；不创建测试文件

**Dependencies:**
- Task 1 已产生 Skill 名称、安装单元和实际工作流；不在 README 发明与 Skill 不同的状态或调用方式。

**Interfaces:**
- Consumes: Task 1 的 `dida-kanbanloop-skill` 名称、三列顺序、单任务调用、子任务后缀和人工重入语义。
- Produces: 面向首次用户的安装/配置/调用/审查/故障说明，供 AC-01、AC-03、AC-09、AC-12 验证。

- [ ] **Step 1: 证明旧 README 不满足新 onboarding 验收**

Run:

```bash
python3 - <<'PY'
from pathlib import Path
r = Path('README.md').read_text(encoding='utf-8')
assert '# dida-kanbanloop-skill' in r
assert '处理中` → `待审查` → `已完成' in r
assert '待办' not in r
PY
```

Expected: 对当前旧 README 为 FAIL（新标题和精确配置序列尚不存在）；这不是交付检查，实施后由 Step 3 的完整检查验证通过。

- [ ] **Step 2: 替换旧流程并写全首次使用路径**

以 `# dida-kanbanloop-skill` 为标题重写 README，按 Spec §8 的八项依次覆盖：①单父任务与审查概览；②OpenCode 安装；③Dida365/TickTick MCP、认证和权限；④Kanban 列；⑤调用与消歧；⑥子任务和本地进度；⑦审查通过/拒绝；⑧故障排查与端到端验证边界。安装示例固定为把含根 `SKILL.md` 的仓库目录放到 `~/.config/opencode/skills/dida-kanbanloop-skill/` 或用户当前 OpenCode 支持的 Skills 目录，并说明重新加载/新会话；不得声称有自动安装器。

看板配置必须原文展示 `处理中` → `待审查` → `已完成`，说明手工配置且 Skill 不补列/改名；明确写“其他列允许存在，会被忽略且永不修改”，以及空白/近似名称不能替代必需列。调用示例同时给出清单/项目、看板、顶层父任务和消歧方式；说明 child 目标会被拒绝，标题无 active 匹配时需要任务 ID 区分未找到/已结束。子任务段落包含复用活跃及已完成且满足项、`[Agent]`、实际完成、本地进度和失去必要性时保持未完成。审查通过与拒绝分别列出用户动作，拒绝步骤必须明确包含“更新父任务描述”并移回 `处理中`。故障排查必须逐项涵盖 MCP 缺失、权限不足、列不匹配、读取不完整、失败/重试与端到端验证边界。删除所有带或不带反引号的旧 `待办` 流程及“完成顶层任务即自动进入已完成”语义，保留 LICENSE 不变。

Expected: 新用户无需阅读 Spec 即可完成 MCP/认证/权限/列准备、加载 Skill、提供完整目标，理解额外列和子任务规则，并知道审查通过或拒绝后如何操作；Step 3 的八组检查逐项覆盖上述内容。

- [ ] **Step 3: 运行 README 聚焦静态与一致性检查**

Run:

```bash
python3 - <<'PY'
from pathlib import Path
r = Path('README.md').read_text(encoding='utf-8')
s = Path('SKILL.md').read_text(encoding='utf-8')
readme_groups = {
    '1-overview': ('# dida-kanbanloop-skill', '父任务', '待审查'),
    '2-install': ('OpenCode', 'SKILL.md', '~/.config/opencode/skills/dida-kanbanloop-skill/', '重新加载', '新会话'),
    '3-mcp-auth-permission': ('Dida365', 'TickTick', 'MCP', '认证', '权限'),
    '4-columns': ('Kanban', '处理中` → `待审查` → `已完成', '其他列', '忽略', '永不修改', '空白', '近似名称'),
    '5-invocation-disambiguation': ('调用', '清单/项目', '看板', '顶层父任务', '消歧', '任务 ID', '未找到', '已结束'),
    '6-children-local-progress': ('复用', '已完成且满足', '[Agent]', '实际完成', '本地进度', '失去必要性'),
    '7-review-reject': ('审查通过', '拒绝', '更新父任务描述', '处理中', '已完成'),
    '8-troubleshooting-verification': ('故障排查', '权限不足', '读取不完整', '重试', '端到端'),
}
for group, tokens in readme_groups.items():
    missing = [token for token in tokens if token not in r]
    assert not missing, f'README {group} missing: {missing}'
assert '待办' not in r, 'legacy 待办 text must be absent, quoted or not'
assert 'Completed top-level tasks move' not in r
assert 'Active top-level tasks move' not in r
assert 'name: dida-kanbanloop-skill' in s
assert r.endswith('\n')
print('README.md focused static checks: PASS (8/8 onboarding groups)')
PY
```

Expected: 输出 `README.md focused static checks: PASS (8/8 onboarding groups)`，人工通读确认通过/拒绝文字与 Skill 的 AD-8 无矛盾；若不一致，以 UR-4 为准只修改 README，不在 Build 中改 Spec/Plan。

### Task 3: 联合静态检查、场景验收与独立审查交接

**Files:**
- Verify: `SKILL.md`
- Verify: `README.md`
- Audit bookkeeping only: `docs/superpowers/plans/2026-10-27-dida-kanbanloop-skill.md`
- Test: 内联 Python 与 Scenario Fixtures；不创建测试文件

**Dependencies:**
- Task 1 和 Task 2 的聚焦检查均已实际通过；任何中间修订均已完成。

**Interfaces:**
- Consumes: 两个实施文件、Spec AC-01～AC-13、S1～S14。
- Produces: 静态检查原始输出、14/14 场景结论、文件范围证据、实时验证限制和独立审查输入；不产生运行时软件或真实 Dida 变更。

- [ ] **Step 1: 运行合并静态验收**

Run:

```bash
python3 - <<'PY'
from pathlib import Path
import re
paths = [Path('SKILL.md'), Path('README.md')]
texts = {}
placeholder = re.compile(r'(?i)(?<![A-Za-z0-9_])(?:TBD|TODO|FIXME|PLACEHOLDER)(?![A-Za-z0-9_])')
for path in paths:
    text = path.read_text(encoding='utf-8')
    texts[path.name] = text
    assert text.endswith('\n'), f'{path}: missing final newline'
    assert not re.search(r'(?m)[ \t]+$', text), f'{path}: trailing whitespace'
    assert not placeholder.search(text), f'{path}: unresolved placeholder'
s = texts['SKILL.md']
r = texts['README.md']
fm = re.match(r'\A---\n(.*?)\n---\n', s, re.S)
assert fm, 'SKILL.md must start with closed YAML front matter'
front = fm.group(1)
assert len(re.findall(r'(?m)^name: dida-kanbanloop-skill$', front)) == 1
assert re.search(r'(?m)^description:\s*\S.*$', front), 'description must be non-empty'
for token in (
    '处理中', '待审查', '已完成', 'N. <动作> [Agent]', '[Agent]',
    'parentId', 'columnId', 'title', 'content', 'desc', 'list_projects',
    'get_project_with_undone_tasks', 'filter_tasks', 'list_columns',
    'get_task_in_project', 'list_completed_tasks_by_date',
    'update_task', 'create_task', 'complete_task',
    'status = 0', 'status = -1', 'status = 2',
    'create_column', 'update_column', 'move_task',
):
    assert token in s, f'SKILL.md missing {token}'
readme_groups = {
    '1-overview': ('# dida-kanbanloop-skill', '父任务', '待审查'),
    '2-install': ('OpenCode', 'SKILL.md', '~/.config/opencode/skills/dida-kanbanloop-skill/', '重新加载', '新会话'),
    '3-mcp-auth-permission': ('Dida365', 'TickTick', 'MCP', '认证', '权限'),
    '4-columns': ('Kanban', '处理中` → `待审查` → `已完成', '其他列', '忽略', '永不修改', '空白', '近似名称'),
    '5-invocation-disambiguation': ('调用', '清单/项目', '看板', '顶层父任务', '消歧', '任务 ID', '未找到', '已结束'),
    '6-children-local-progress': ('复用', '已完成且满足', '[Agent]', '实际完成', '本地进度', '失去必要性'),
    '7-review-reject': ('审查通过', '拒绝', '更新父任务描述', '处理中', '已完成'),
    '8-troubleshooting-verification': ('故障排查', '权限不足', '读取不完整', '重试', '端到端'),
}
for group, tokens in readme_groups.items():
    missing = [token for token in tokens if token not in r]
    assert not missing, f'README {group} missing: {missing}'
assert '待办' not in r, 'legacy 待办 text must be absent, quoted or not'
assert 'Completed top-level tasks move' not in r
assert 'Active top-level tasks move' not in r
print('Combined static acceptance: PASS (2 files; README 8/8 groups)')
PY
```

Expected: 输出 `Combined static acceptance: PASS (2 files; README 8/8 groups)`；该结果只证明静态契约，不能替代 Step 2 或实时 MCP 证据。

- [ ] **Step 2: 逐条走查 S1～S14 并核对验收 ID**

对 Scenario Fixtures 的每一行，在实施报告中记录：约束它的 `SKILL.md` 章节、预期写入序列、明确禁止的调用、最终父/子状态和报告措辞，以及覆盖的 AC ID。不得调用真实 MCP。若冲突来自 `SKILL.md`/`README.md` 对既定 Spec 的错误表达，Build 只修这两个文件并重跑 Step 1 与失败场景；若需要改变 Spec/Plan、任务、接口或验收，立即停止并交回 Tower 的 pre-Build review loop，不得在 Build 中编辑它们。

Expected: 14/14 均有明确 PASS；特别是 S3 的额外列允许但空白/近似替代为零写入，S4/S13 双分支不完整即停止，S7 无子任务回移，S8 无父完成，S9 含 ID/status/pending，S10 无盲目创建重试，S12 区分未找到/已结束，S14 拒绝 child 父目标。记录任何修订，不以“整体看起来符合”替代逐场景证据。

- [ ] **Step 3: 核对变更范围与 Markdown 基本完整性**

Run:

```bash
git diff --check
git status --short --untracked-files=all
```

Expected: `git diff --check` 无输出且退出 0。把最新输出原文记录为 `post_build_status`，并与 Task 1 的 before 路径集合比较：`after_paths - before_paths` 只能是 `{SKILL.md, README.md}` 的子集；不得出现其他新增路径或删除既有路径。基线中的每个未跟踪路径必须原样保留并单独列出，不能因“预期只应有四份文档”而清理。 tracked 实施改动仅限 `README.md`；Spec 不得变化，Plan 只能有 Audit Record 簿记，`LICENSE` 不得变化。发现任何越界或未知路径时停止并交 Tower，不清理、不回退、不 stage/commit。

- [ ] **Step 4: 记录验证层级与未验证项**

在交付/审计记录中分开列出：Task 1/2/3 静态检查实际结果、场景通过数、是否做过真实 MCP 端到端测试。本计划没有专用实时夹具，不执行真实 Dida 写入；除非另有明确授权的测试任务和证据，固定记录“未做实时 MCP 端到端验证”。

Expected: 验证声明与实际命令/走查一致；没有把关键词 PASS 写成运行时 PASS，也没有隐瞒实时验证缺口。

- [ ] **Step 5: 交给全新上下文做独立 post-Build 审查**

向独立只读审查者提供原始 UR/DC、完整 Spec、**完整 Plan 及其 Audit Record**、两个实施文件、`pre_build_status`/`post_build_status` 与路径集合比较、Step 1/3 原始输出、Step 2 全场景记录和 Step 4 限制。要求逐项核对 AC-01～AC-13、六维质量（需求完整性、逻辑、边界、文档质量、测试有效性、真实运行证据）并返回 PASS 或具体 BLOCKER/MAJOR/MINOR；审查者不得修改文件。

Expected: 审查者保持只读并返回独立 verdict、问题计数和证据，由 Tower 或构建会话把原样结论簿记到 Audit Record。Hard 任务交付前必须为 PASS；发现问题时由 Tower 分派聚焦修复并开启新的独立复审，按 `review_round:N/3` 计数，构建者不得自审代替。

## Acceptance Traceability Matrix

记法：`T1`/`T2`/`T3` 分别指 Task 1/2/3，`Tn-Stepk` 指 Task n 的 Step k；`Sn` 指上方 Scenario Fixture n。例如 `T1-Step3` 是实施步骤，`S3` 是行为场景，二者不可混用。

| AC | 实施任务 | 必需的验证证据 |
|---|---|---|
| AC-01 | T1-Step2/3，T2-Step2 | T1/T2 聚焦检查；T3-Step1 front matter/安装路径；T3-Step3 基线与文件范围 |
| AC-02 | T1-Step3 | T3-Step2 的 S1/S2/S3/S12/S14；child 目标、无 active 标题匹配均有零写入证据 |
| AC-03 | T1-Step2/3，T2-Step2 | 两文件精确列名/顺序与额外列说明；S3 额外列允许、空白/近似替代及列写入禁令 |
| AC-04 | T1-Step3 | S4 的 `filter_tasks` + completed 双分支、空字段/abandoned/顺序证据；S13 capped/截断/日期缺口即 `W0`；独立语义审查 |
| AC-05 | T1-Step3 | S5 首次写入账本、父 `taskId + columnId + status = 0` 回读和已在 `处理中` 变体 |
| AC-06 | T1-Step3，T2-Step2 | S6 活跃复用、已完成且满足项不重建、字段白名单、失去必要性与本地进度；S7 有真实证据才完成；静态 `N. <动作> [Agent]` 检查 |
| AC-07 | T1-Step2/3 | S7 完成前身份复核、仅单项完成、列前后一致、异常不回移；S14 child 父目标零写入 |
| AC-08 | T1-Step2/3 | S8 只到 `待审查`、active、停止及 abandoned/用户排除项报告；S12 与全场景无父完成/自动 `已完成` |
| AC-09 | T1-Step3，T2-Step2 | S10 中断去重、S11 新描述修正循环、S12 未找到/已结束与审查状态零写入；README 人工步骤 |
| AC-10 | T1-Step3 | S1/S2/S3/S4/S9/S10/S13 的停止条件、双分支完整性、限流退避、重试上限和部分成功账本 |
| AC-11 | T1-Step3 | S5 首次父移动、S6 创建、S7 child 完成、S8 最终移动的响应/回读；S9 confirmed `taskId + status` 与 pending actions；S10 reconciliation；缺证据不得报成功 |
| AC-12 | T2-Step2/3 | README 8/8 聚焦检查组、额外列说明、任何带或不带反引号的旧 `待办` 文本缺席、首次用户路径独立审查 |
| AC-13 | T1-Step1，T3-Step1～5 | pre/post status 与路径集合比较、合并静态输出、14/14 场景记录、实时验证限制、完整 Plan/Audit Record 交接和独立 post-Build PASS |

## Execution Handoff

1. **当前下一责任人**：Tower 分派全新只读上下文做 round 2 Spec/Plan pre-Build re-review；审查者返回 verdict/证据，Tower 把原样结论写入 Audit Record。W3 continuation 自查不能替代，W4 的 REVISE 不能当作本轮结论。
2. **Build**：仅在 round 2 pre-Build PASS 后，先记录 status 基线，再按 Task 1 → Task 2 → Task 3 顺序执行。任务间有明确接口依赖，不并行写两个文档；默认交给轻量实施层，除非满足既有升级规则。
3. **Fix/Audit**：Build 中的表达性修复只能改 `SKILL.md`/`README.md`，修复后重跑相关静态检查和失败场景，再由全新上下文复审。任何需要改 Spec/Plan、任务、接口或验收的发现都停止 Build 并交回 Tower 的 pre-Build review loop，不得由 Build 自行扩权。
4. **交付**：只在静态检查、14/14 场景、pre/post 基线与范围检查和独立 post-Build PASS 都有实际证据后交付；没有专用实时夹具时继续明确“未做实时 MCP 端到端验证”。不自动提交或推送。
