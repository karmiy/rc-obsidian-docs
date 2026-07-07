# 03 导入与版本方案

更新时间：2026-07-03

## 目标

支持从 URL / Folder 导入 Skill Set。导入不是直接入库，而是先生成 preview，用户确认后保存。

## 标准 Bundle

Skill Set 标准结构至少包含：

```text
README.md
skills/
  <skill>/
    SKILL.md
```

允许额外文件：

```text
assets/*
references/*
docs/*
```

`manifest.json` 不要求出现在 import output 中。它由后端根据数据库内容生成。

## Import 流程

1. 用户在 New Skill Set dialog 选择 URL 或 Folder。
2. 前端调用 analyze job endpoint 启动分析，接口立即返回 `jobId` 和初始状态；前端按 `jobId` 轮询状态。
3. 系统获取 source：
   - URL：后端让 remote daemon 把 GitHub / GitLab 仓库 clone 到当前 import job 的 sandbox `source/`。
   - Folder：前端通过本地 daemon 读取用户选择的本地文件夹，并把 source file tree 上传给后端；后端再通过 remote daemon 写到 sandbox `source/`。
4. 后端通过 system runtime 调用 remote daemon 上的 Claude Code。
5. Claude Code 自己分析 `source/`，并按 AI Desk Skill Set 标准写 bundle 到 `output/`。
6. 后端通过 remote daemon file API 扫描并读取 `output/`。
7. 后端只校验 `output/` 是否符合 Skill Set 标准并生成 preview，不对 source 是否已有 `SKILL.md`、`skills/` 或标准目录做前置业务判断。
8. job 进入 `completed` 后，前端把 preview 写入 draft store 并展示 metadata review。
9. 用户确认后入库。
10. 成功、失败或取消都清理 sandbox。

## Analyze Job / Polling / Stop

New Skill Set 的 URL / Folder import preview 采用 job 模型，避免长 HTTP 请求绑定 UI：

```text
POST /skill-sets/import-url/analyze
POST /skill-sets/import-folder/analyze
GET  /skill-sets/import-jobs/{jobId}
POST /skill-sets/import-jobs/{jobId}/cancel
```

Job 状态：

- `queued`
- `analyzing`
- `completed`
- `failed`
- `cancelled`

Job 状态必须存到 Redis 共享 store，而不是 Python backend 进程内 dict。Redis value 至少包含：

- `jobId`
- `status`
- `executionId`
- `sandboxRoot`
- `source`
- `preview`
- `error`
- owner scope，例如 `accountId` / `extensionId`
- `createdAt` / `updatedAt`

`failed.error` 必须保留 bounded diagnostic detail，例如 runtime 执行异常类型、daemon 返回的 stderr/stdout 摘要、output 读取失败原因或 output 文件数量。不能只返回统一的 `Skill Set conversion failed in system runtime`，否则长任务失败后无法判断原因。

Redis key 设置 15 分钟 TTL。后台 Python `asyncio.Task` 可以保留为当前 backend 实例内的取消优化，但 poll/cancel 的正确性只能依赖 Redis 中的 job 状态和 daemon `executionId`。

Cancel 流程：

1. 从 Redis 读取 job 并校验 owner。
2. 如果 job 未结束，先写 `cancelled`，防止后台 worker 后续覆盖状态。
3. 使用 Redis 中的 `executionId` 调用 remote daemon cancel execution。
4. 使用 Redis 中的 `sandboxRoot` 清理当前 job sandbox。
5. 如果 cancel 请求刚好打到启动 job 的 backend，可顺手 cancel 本地 `asyncio.Task`；如果打到其他 backend，也不能影响 cancel 正确性。

Worker 写 `completed` / `failed` 前必须先读取当前 Redis 状态。如果已经是 `cancelled`，不得覆盖为 `completed` 或 `failed`。

前端行为：

- 点击 Import 后 primary 按钮显示 disabled `Analyzing...`。
- 前端按固定 3 秒间隔轮询 import job status。
- 进行中 footer 左侧显示 text-style `Cancel` 按钮；点击后只显示 disabled `Canceling...`，隐藏 `Analyzing...` 按钮。
- cancel 调用 cancel endpoint，后端取消 daemon execution 并删除当前 job sandbox。
- cancel 完成后保持在当前 import 页面，不回到一级 chooser，并清空当前表单 / preview / job 状态。
- 普通关闭弹框不清空 draft；重新打开仍能看到当前 job / preview 状态。
- preview 阶段点击 footer `Cancel` 清空 draft，让用户可以重新 import。

当前实现保留旧同步 `/preview` endpoints 作为兼容路径；New Skill Set dialog 使用 analyze job endpoints。Sync preview 仍走现有同步接口，后续如需 Stop sync，可复用同一 job service。

## Source 处理原则

Skill Set import 的核心是让 Claude Code 理解任意仓库并转换成标准 Skill Set bundle。后端不应该把普通 Skill import 的扫描规则套到 Skill Set import 上。

后端不做这些前置限制：

- 不要求 source 中存在 `SKILL.md`。
- 不要求 source 中存在 `skills/`。
- 不因为 source 不是标准 Skill Set 结构就返回空 files 或直接失败。
- 不按业务语义提前判断哪些源码文件“像 skill”。
- 不设置 source file count / file size 级别的业务限制来决定是否允许 Claude 分析。

后端只负责：

- 准备 sandbox `source/`。
- 调用 Claude Code converter。
- 读取并校验 `output/`。
- 做路径安全和 cleanup。

## Sandbox

Sandbox path：

```text
<system_remote_runtime_cwd>/tmp/skill-set-imports/<job_id>/
  source/
  output/
  logs/
```

当前本地 fallback 示例：

```text
/Users/karmiy.hong/repos/ai-desk-system-runtime/tmp/skill-set-imports/<job_id>/
```

Sandbox 必须位于 Claude Code runtime 可访问的 filesystem。system runtime 实际执行位置是 remote daemon，不是 Python backend 进程；backend 只负责准备输入、调用 remote daemon、读取输出。

如果 backend 和 remote daemon 不共享 filesystem，实现必须通过 remote daemon file API 写入和读取：

- URL source：使用 remote daemon 的 generic repo clone 能力，把 repo clone 到 `source/`
- Folder source：使用 `write_file_tree(source_path, files, replace=true)` 把前端上传的 source files 写到 `source/`
- `/api/v1/agent/execute/stream` / `execute_agent_stream(cli_type="claude", cwd=sandbox_root, ...)`：让 remote daemon 执行 Claude Code，并由 backend 收集 stream complete/final response
- `list_files(output_path, depth=...)` + `read_file(path)`：把 `output/` bundle 拉回 backend；图片、压缩包、字体等二进制文件必须用 daemon file read 的 base64 模式读取，并在 Skill Set file payload 中标记 `encoding=base64`
- `DELETE /api/v1/workspace` / `remove_workspace(sandbox_root)`：清理 import job sandbox

当前生产环境只有一个 remote daemon，因此同一个 import job 的 `write_file_tree`、`execute_agent_stream`、`list_files/read_file` 会落在同一个 daemon filesystem。未来如果 remote daemon 扩展为多副本，import job 必须绑定同一个 daemon instance，不能依赖 Kubernetes service 随机路由。

## System Runtime 调用链

Skill Set import preview 由后端 job service 包装成可轮询、可取消的 analyze job；前端不直接消费 runtime stream。后端内部调用 remote daemon 的 agent stream 接口，等待 stream `complete` / final response 后再读取 `output/`，避免长时间 converter 任务被普通 HTTP response write timeout 截断。

Converter 调用 remote daemon 时使用 backend 全局 `REMOTE_DAEMON_TIMEOUT`，不要额外硬编码 10 分钟之类的更短 cap。大型仓库可能需要超过 10 分钟才能由 Claude 转成标准 Skill Set bundle；active job 轮询期间如果 Redis TTL 低于 5 分钟，续回 15 分钟，避免分析仍在运行但 job state 先过期，同时避免每 3 秒 poll 都写 Redis。

后端执行顺序：

```text
1. 创建 import job sandbox path，并生成 daemon `execution_id`
2. URL source：调用 remote daemon repo clone，把仓库 clone 到 sandbox/source
3. Folder source：调用 remote daemon write_file_tree，把前端上传的文件写入 sandbox/source
4. 调用 remote daemon execute_agent_stream，透传 `execution_id`，cwd=sandbox_root，让 Claude Code 分析 sandbox/source 并写 sandbox/output
5. backend 监听 stream chunk，按 final response 或 complete exit_code 判断 Claude CLI 是否成功退出
6. 返回成功后，调用 remote daemon list_files/read_file 读取 sandbox/output
7. 读取并消费 `output/__metadata.json`，再校验 output 并生成 preview
8. cleanup sandbox：调用 remote daemon `remove_workspace(sandbox_root)`
```

`execute_agent_stream` 会让 remote daemon 执行类似：

```bash
claude --print --verbose --output-format stream-json --dangerously-skip-permissions --model "<model>" "<prompt>"
```

daemon 会在 Claude CLI 运行期间持续通过 WebSocket 发送 stdout/stderr/error/complete chunk，并在结束时发送 final response。backend 以 final response 或 complete chunk 的 `exit_code` 作为 Claude 已结束的信号；如果失败，需要把 bounded stdout/stderr/error/exit_code 写入 import job error diagnostic。

Cancel / cleanup 只允许删除当前 job 的 sandbox root：

```text
<system_remote_runtime_cwd>/tmp/skill-set-imports/<job_id>
```

不要删除 `<system_remote_runtime_cwd>` 或 `<system_remote_runtime_cwd>/tmp/skill-set-imports` 这种父目录。remote daemon 的 `file/delete` 只适合单文件；递归清理目录应复用 `remove_workspace`。

## Claude Code Contract

Claude Code 把文件写到：

```text
output/
```

不需要通过 stdout JSON 返回全部文件内容。

最小输出：

```text
output/
  __metadata.json
  README.md
  skills/
    <skill>/
      SKILL.md
```

`__metadata.json` 是 converter 给 backend 的控制文件，不会进入最终 Skill Set bundle。格式：

```json
{
  "name": "Human readable Skill Set name",
  "description": "One plain-text sentence without markdown or HTML."
}
```

backend 生成 preview metadata 时优先使用 `__metadata.json` 中非空的 `name` / `description`；如果 Claude 没输出、JSON 无效、字段为空，才 fallback 到 backend 根据 source URL、README heading、README 可读摘要计算。README 摘要 fallback 必须跳过 HTML 装饰行、badge/image/link-only 行和 markdown heading，避免把 `<p align="center">` 这类 raw HTML 作为 description。

Claude 可以额外写入它判断对 set 有用的文件：

- `assets/*`
- `references/*`
- docs
- skill supporting files

其中二进制 asset（例如 `png`、`jpg`、`gif`、`webp`、`ico`、`pdf`、`zip`、字体文件）在 backend preview/confirm payload 中必须保持 `encoding=base64` + `contentBase64`，不能作为 utf8 `content` 保存。

Claude Code 可以基于自己的判断选择分析方式，包括在 sandbox 内运行必要脚本或命令来理解 source。约束是：

- 所有操作必须限制在 import job sandbox 内。
- 最终产物必须写到 `output/`。
- 后端只信任并校验 `output/`，不直接信任 source repo 原始结构。
- 如果 source 内容没有有效信息，Claude 应输出一个空 preview bundle，而不是强行编造 skill。
- 不要把 dependency/vendor/cache/build output 写入 `output/`，除非它是手写且对 Skill Set 使用必要的少量文件。

无效 source 的建议输出：

```text
output/
  README.md
  skills/
```

其中 `README.md` 说明无法从 source 中识别有效 Skill Set 内容。后端返回 preview，让用户看到这是无效或空导入。

## Import Source 场景

### Source Acquisition

URL import、local folder import、URL sync、local folder sync 共用同一个 source normalize / converter / preview / confirm 流程，差别只在 source tree 如何进入后端。

URL source：

- `origin_type = url`
- `origin_url` 保存用户输入的 source URL。
- import preview / sync preview 时由后端让 remote daemon clone GitHub / GitLab repo 到 sandbox `source/`。
- 如果 URL 指向 repo 的 subtree，Claude Code 的 source path 应指向该 subtree；repo 仍可以完整 clone 到 sandbox 内，便于 Claude 根据需要参考上下文。

Local folder source：

- `origin_type = local_folder`
- `origin_url` 保存用户选择的本地绝对 folder path。
- import preview 时前端通过本地 daemon 读取 folder，把 source files 上传给后端。
- sync preview 时前端再次按保存的 folder path 调用本地 daemon 读取 folder，把新的 source files 上传给后端。
- backend / remote daemon 不直接访问用户机器上的 local path；它只处理前端上传的 file tree。
- 如果本地路径不存在、权限不足或用户取消读取，前端应展示可恢复错误，不创建 preview job。

所有 source 形态都走 Claude Code converter：

- 已经是标准 Skill Set bundle 的 source，也由 Claude 复制/整理到 `output/`。
- 只有 `skills/` 的 source，由 Claude 判断是否生成 `README.md` 并整理 skills。
- 没有 `skills/` 的 source，由 Claude 分析是否可以构造 skills。
- 如果无法构造，Claude 输出空 preview bundle，让用户在 preview 中确认 source 无有效内容。

## Preview 展示

Import preview 和 sync preview 共用同一个前端 preview 组件，例如 `SkillSetBundlePreviewDialog`。

主弹框展示解析完成状态：

- metadata form，用于确认或修改 `name`、`description`、`visibility` 等字段
- form 默认使用后端解析出的 metadata 预填，例如从 README、repo name、source metadata 推断 `name` 和 `description`
- 即将分配的 version
- sub skills
- set-level files
- 文件数量

主弹框提供 `Review files` 按钮，打开二级 preview 弹框。二级弹框复用 `Set Content` 的 file tree/editor 组件，但内容只读。

二级 preview 弹框：

- 左侧 file tree 每个节点带 checkbox，默认全选。
- 右侧显示当前选中文件内容，read-only。
- 勾选目录等于选择目录下全部文件。
- 取消勾选目录等于排除目录下全部文件。
- 用户取消勾选的文件或目录不会进入 confirm payload。
- 用户可以用它排除不希望作为 set 一部分的文件。
- 必须保留 `README.md` 和至少一个 `skills/<skill>/SKILL.md`，否则 confirm disabled。

Confirm request 应携带用户最终选择的 file paths：

```json
{
  "importJobId": "job-id",
  "metadata": {
    "name": "Superpowers",
    "description": "...",
    "visibility": "public"
  },
  "selectedPaths": [
    "README.md",
    "skills/brainstorming/SKILL.md"
  ]
}
```

Preview 不写 DB。

Sync preview 也遵循同样规则：

- 点击 `Set Metadata` 的 `Sync from origin` 后生成 sync preview。
- 主弹框展示 metadata form，默认用重新解析出的 metadata 预填；用户确认前可以修改。
- 使用同一个 `Review files` 二级只读 file tree 弹框。
- 用户取消勾选的文件或目录不会进入 sync confirm payload。
- 用户确认后，后端用用户勾选后的 selected bundle 全量替换当前 Skill Set 内容。
- Sync confirm 是物理替换：旧的 `SkillSetBundleFile`、owned `Skill`、owned `SkillFile` 数据删除后重新写入。
- Sync confirm 成功后统一 bump MINOR，例如 `1.0.31 -> 1.1.0`。

## Versioning 规则

后端拥有 `SkillSet.version`。

初始版本：

```text
1.0.0
```

格式：

```text
MAJOR.MINOR.PATCH
```

采用 SemVer-style 规则：

- PATCH：用户在 `Set Metadata` 保存 metadata，或在 `Set Content` 保存文件内容/文件结构
- MINOR：用户执行 `Sync from origin` 并确认保存
- MAJOR：破坏性结构变化或不兼容行为变化

进位规则：

- bump PATCH：`1.0.0` -> `1.0.1`
- bump MINOR：`1.0.9` -> `1.1.0`
- bump MAJOR：`1.9.9` -> `2.0.0`

Import preview 可以展示 source version，如果 source 中能识别到。但最终持久化的 AI Desk version 由后端分配。

本期 UI 不让用户选择 PATCH / MINOR / MAJOR：

- import 新 set：`1.0.0`
- metadata save：bump PATCH
- content save：bump PATCH
- sync confirm：bump MINOR
- MAJOR 保留给未来 admin/manual breaking-change release

## 校验

以下情况拒绝 import output：

- 缺少 `README.md`
- 没有 `skills/<skill>/SKILL.md`
- unsafe paths
- duplicate paths
- unsupported binary encoding

对于前端上传的 local folder source，以及 Claude Code 输出的 `output/`，需要做 ignored paths 过滤。过滤按 path segment 生效，不只匹配根目录。

必须忽略：

- 任意 path segment 等于 `node_modules`，大小写不敏感，例如 `node_Modules`
- 任意 path segment 以 `.` 开头，例如 `.git`、`.cache`、`.next`、`.DS_Store`
- 任意 path segment 以 `__` 开头，例如 `__pycache__`

还应忽略常见 dependency / generated / cache output：

- `dist`
- `build`
- `coverage`
- `out`
- `target`
- large binaries
