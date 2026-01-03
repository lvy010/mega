# Orion Server Buck2 日志架构

## 架构

采用每个 target 独立触发 Buck2 构建并生成专属日志文件的方案，天然实现多 target 日志隔离。调度器在 Task 层聚合各 target 的状态和结果，完成 CL 级状态汇总。直接复用现有 REST/WS 接口，前端能够获取 target 级的实时状态与日志信息。

- **CL (Change List)** -> **Task** (1:N)：一个代码变更可多次提交任务（重试、新增 target 等）
- **Task** -> **Build** (1:N)：一次任务提交可包含多个 Build，每个对应一个 Buck2 target
- **日志隔离**：以 Build 为粒度隔离，Task 级别通过聚合得到总体状态
- **Target 区分**：通过 Build 的 `target` 字段标识具体构建目标

```
CL (cl_id: 123)
│
├── Task 1 (task_id: 01JN1111-...) ← 第一次提交
│   ├── Build A (build_id: 01JN2222-..., target: "//app:libA")
│   │   ├── 状态: Completed, exit_code: 0
│   │   ├── 日志: {task_id}/repo/{build_id}.log
│   │   └── 接口: /task-output/{build_id}, /task-history-output
│   ├── Build B (build_id: 01JN3333-..., target: "//app:libB")
│   │   ├── 状态: Building, exit_code: null
│   │   ├── 日志: {task_id}/repo/{build_id}.log
│   │   └── 接口: 同上
│   └── Build C (build_id: 01JN4444-..., target: "//app:libC")
│       ├── 状态: Failed, exit_code: 1
│       ├── 日志: {task_id}/repo/{build_id}.log
│       └── 接口: 同上
│
└── Task 2 (task_id: 01JN5555-...) ← 重试 libC 或新增 target
    └── Build C' (build_id: 01JN6666-..., target: "//app:libC")
        ├── 状态: Completed, exit_code: 0
        ├── 日志: {task_id}/repo/{build_id}.log (新的日志文件)
        └── 接口: 同上
```

说明：

- CL/Task 聚合：任一 Build 失败/中断/取消 → Task 失败；有运行中则 Building；全成功则 Completed；partial_success 在"有成功且有运行中或失败"时为 true。
- 回退：未提供 target 时降级为 `//...`，不中断构建，并记录警告。

## 后端更新

- ~~创建任务时 `builds` 数组可提供可选 `target`：提供 `target`：worker 仅构建该目标，日志/状态归属该 Build。未提供：自动解析目标列表（兼容旧行为），并记录回退。（Orion模块已实现）~~
- 日志 key 结构：`{task_id}/{repo_last_segment}/{build_id}.log`，Build 级别完全隔离；`BuildDTO.log_path` 返回该路径，便于前端显示/下载。
- Build 状态推导：运行中 -> `Building`；未结束且不在 active -> `Pending`；结束且 `exit_code==0` -> `Completed`；`exit_code` 为其他值 -> `Failed`；`exit_code` 缺失 -> `Interrupted`；`Canceled` 预留支持。
- Task 聚合：
- 优先级：存在 `Failed/Interrupted/Canceled` -> `Failed`; 其次存在 `Building/Pending` -> `Building`; 否则全为成功 -> `Completed`; 无数据 -> `NotFound`。
- `partial_success=true` 当至少一个 Build 成功且仍有运行中或失败的 Build。

## HTTP 接口

- 创建任务 `POST /task`
- Request Body（摘录）：
- `repo` (string)
- `cl_link` (string)
- `cl` (number)
- `builds`: `BuildRequest[]`
- `buck_hash` / `buckconfig_hash`
- `args?`
- `target?` 可选 Buck2 label（如 `//app:server`）
- Response: `task_id` 与每个 build 的 `build_id`/状态（queued/dispatched/error）。
- 查询 CL 任务列表 `GET /tasks/{cl}`
- Response: `TaskInfoDTO[]`
- `status`: Task 聚合状态
- `partial_success`: 是否部分成功
- `build_list`: `BuildDTO[]`（含 `status`, `target`, `id`, `output_file`, `log_path` 等）
- 实时日志 `GET /task-output/{build_id}` (SSE)
- 事件类型 `log`，数据为该 Build 的行输出；按 Build 隔离。
- 历史日志 `GET /task-history-output`
- Query: `task_id`, `build_id`, `repo`, `start?`, `end?`
- 返回 `data: string[]` 与 `len`。

## WebSocket（Worker）

- 服务器下发 `Task` 消息包含 `target: Option<String>`（后端解析/回退后传入）。
- Worker 回传 `BuildOutput`、`BuildComplete`、`TaskPhaseUpdate` 不变。

## 前端

- 按 CL 查询后，使用 `TaskInfoDTO.status` 显示总体状态，若 `partial_success=true` 显示"部分成功/仍在构建"提示。
- Build 级列表用 `build_list` 渲染；点击某 Build 使用其 `id` 订阅 SSE 或读取历史日志，仅显示该 target 的日志。
- 高亮：`status` 为 `Failed/Interrupted/Canceled` 的 Build，日志中可配合后端已解析的 `cause_by` 字段做折叠/展开。
- 兼容：未提供 `target` 的旧请求依旧有效，UI 继续依赖 `target` 字段作为标签；字段新增均向后兼容。

#### 细化

- `/tasks/{cl}`：返回聚合状态和按 target 拆分的 `build_list`（含 `target`、`status`、`log_path`）。
- `/task-output/{build_id}`（SSE）与 `/task-history-output`：按 build_id/target 读取，日志天然隔离。
- UI 仅需用 `build_id`/`target` 做筛选与高亮失败条目，`partial_success` 用于"部分成功/仍在进行"提示。

## 测试
1) 创建任务：`POST /task`，`builds` 数组中放多个 target（示例：`//app:libA`、`//app:libB`）。未传 target 会回退 `//...` 并告警但不中断。
2) 查看聚合：`GET /tasks/{cl}`，应看到按 target 拆分的 `build_list`，以及聚合 `status`/`partial_success`。
3) 日志隔离：分别订阅 `/task-output/{build_id_of_libA}` 与 `/task-output/{build_id_of_libB}`，只应看到各自 target 日志；或用 `/task-history-output` 分别读取。
4) 回退兼容：提交一个未带 target 的 build，确认仍能返回 build_id 并可读取日志。

