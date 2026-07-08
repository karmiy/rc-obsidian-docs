# Skill 目录软链接与版本更新机制验证报告

日期：2026-07-08  
测试项目：

- `/Users/karmiy.hong/Documents/Projects-stage-2/Fiji`
- `/Users/karmiy.hong/Documents/Projects-stage-2/FIJI2`

## 1. 摘要

本报告验证 Codex、Claude Code、Cursor、Copilot 四个 CLI 对 project skills 目录软链接、skill 命名、重复 name、链式引用、同 session 版本更新的处理差异。

核心结论如下：

| 主题 | 结论 |
|---|---|
| 统一入口 | `.agents/skills` 不能作为四个 CLI 的统一入口；Claude Code 未加载该目录 |
| 专属入口 | `.codex/skills`、`.claude/skills`、`.cursor/skills`、`.github/skills` 均支持软链接 |
| 目录名与 `name` | 建议保持一致；Cursor 在目录名与 `name` 不一致时未触发链式 skill |
| 重复 `name` | Copilot 按 `SKILL.md` 的 `name` 去重；Codex 不去重；Claude Code 和 Cursor 更依赖目录名 |
| 独立 symlink | 每个 skill 子目录单独作为 symlink 时，四个 CLI 均可加载 |
| symlink 切版本 | 同一 session 内从 `1.2.0` 切到 `1.3.0` 后，Codex 仍可能绑定旧 realpath，`session/load` 也不能保证刷新 |
| 稳定路径原地更新 | 方案 A 对 Codex 有效：稳定路径不变、原地替换 `SKILL.md` 后，同一 session 可读到新内容 |
| workspace 中项目目录为 symlink | 四个 CLI 均能沿 `FIJI` symlink 进入真实项目；未出现将 workspace 误判为空目录的情况 |

## 2. 测试方法

### 2.1 CLI 与入口目录

| CLI | Project skills 目录 |
|---|---|
| Codex | `.codex/skills` |
| Claude Code | `.claude/skills` |
| Cursor | `.cursor/skills` |
| Copilot | `.github/skills` |

### 2.2 判定原则

1. 优先检查 CLI 初始化事件、ACP stream event、tool 调用事件和模型首轮可见结果。
2. 区分“初始化已加载”和“后续被模型主动搜索/读取”。
3. 版本更新实验仅使用 ACP `session/prompt` 或 ACP `session/load`，不使用常规 stdout `--resume`。
4. 本报告不声称还原底层 provider API request payload；结论基于可观测事件链。

## 3. 实验一：`.agents/skills` 统一入口软链接

### 实验目的

验证 `.agents/skills` 是否可作为四个 CLI 的统一 project skills 入口。

### 预期

如果 CLI 支持 `.agents/skills`，新 session 初始化或首轮上下文中应直接可见测试 skill，且不需要后续目录搜索。

### 实验样板

在 Fiji 项目下将 `.agents/skills` 指向外部临时 skills 目录，并创建测试 skill。

### 实验步骤

1. 在外部目录创建测试 `SKILL.md`。
2. 将 Fiji 项目的 `.agents/skills` 设置为指向该目录的 symlink。
3. 分别启动四个 CLI 新 session。
4. 询问“你有哪些 skill”。
5. 检查 stream/session event 是否有后续搜索目录或读取文件行为。

### 判定标准

首轮即可见测试 skill，且未观察到后续目录搜索或读取文件，判定为初始化加载成功。

### 实验结果

| CLI | 是否加载 `.agents/skills` | 证据 |
|---|---:|---|
| Codex | 是 | 首轮可见测试 skill，未观察到后续目录搜索或文件读取 |
| Claude Code | 否 | `system init` 的 `skills` / `slash_commands` 中未出现测试 skill |
| Cursor | 是 | 首轮可见测试 skill，未观察到后续目录搜索或文件读取 |
| Copilot | 是 | `session.skills_loaded` 中出现测试 skill |

### 实验结论

`.agents/skills` 不能作为四个 CLI 的统一入口。Claude Code 不加载该目录。

## 4. 实验二：各 CLI 专属 skills 目录软链接

### 实验目的

验证四个 CLI 是否能从各自专属 project skills 目录加载 symlink 目标。

### 预期

四个 CLI 均应在初始化阶段或首轮上下文中看到专属目录 symlink 指向的外部 skills。

### 实验样板

在 Fiji 项目下分别将以下目录设置为指向同一外部 skills 目录的 symlink：

| CLI | Project skills 目录 |
|---|---|
| Codex | `.codex/skills` |
| Claude Code | `.claude/skills` |
| Cursor | `.cursor/skills` |
| Copilot | `.github/skills` |

### 实验步骤

1. 在外部目录创建测试 skill。
2. 分别设置四个专属 skills 目录 symlink。
3. 分别启动四个 CLI 新 session。
4. 询问可用 skill。
5. 检查初始化事件、首轮回答和后续文件访问事件。

### 判定标准

首轮可见测试 skill，且未依赖后续主动搜索，判定为加载成功。

### 实验结果

| CLI | 是否加载专属目录 symlink | 初始化证据强度 |
|---|---:|---|
| Codex | 是 | 中 |
| Claude Code | 是 | 高 |
| Cursor | 是 | 中 |
| Copilot | 是 | 高 |

证据说明：

| CLI | 证据 |
|---|---|
| Codex | stream 未展开 hidden prompt，但首轮回答精确列出测试 marker 与路径，且无工具事件 |
| Claude Code | `system init` event 直接包含 `skills` / `slash_commands` |
| Cursor | 首轮回答精确列出 `.cursor/skills/...` 路径，且无工具事件 |
| Copilot | 用户消息前出现 `session.skills_loaded` event |

### 实验结论

四个 CLI 均支持通过各自专属 project skills 目录加载 symlink 目标。跨 CLI 兼容时应分别提供 `.codex/skills`、`.claude/skills`、`.cursor/skills`、`.github/skills`。

## 5. 实验三：相同 `name`、不同目录名的处理

### 实验目的

验证多个 skill 的 `SKILL.md` frontmatter `name` 相同、目录名不同时，各 CLI 如何展示或去重。

### 预期

如果 CLI 以 `SKILL.md` 的 `name` 为主，可能出现同名展示或去重；如果以目录名为主，则会显示不同目录名。

### 实验样板

外部 skills 目录包含 3 个 skill：

| 目录名 | `SKILL.md` name |
|---|---|
| `zz-fiji-test` | `zz-fiji-test` |
| `zz-fiji-test-123123123` | `zz-fiji-test` |
| `zz-fiji-test-abcdefg` | `zz-fiji-test` |

三者 `SKILL.md` 内容完全一致。

### 实验步骤

1. 将三个目录放入外部 skills 目录。
2. 分别通过四个 CLI 的专属 skills 目录 symlink 加载。
3. 启动新 session。
4. 询问可用 skill。
5. 检查展示名、去重行为和 event。

### 判定标准

以初始化或首轮可见的 skill 列表为准，记录展示名和数量。

### 实验结果

| CLI | 观察结果 |
|---|---|
| Codex | 可见 3 条记录，3 条均显示为 `zz-fiji-test`，通过不同路径区分 |
| Claude Code | 初始化 inventory 中出现 3 条记录，名称分别为 `zz-fiji-test`、`zz-fiji-test-123123123`、`zz-fiji-test-abcdefg` |
| Cursor | 可见 3 条记录，名称分别为 `zz-fiji-test`、`zz-fiji-test-123123123`、`zz-fiji-test-abcdefg` |
| Copilot | `session.skills_loaded` 中只出现 1 条 `zz-fiji-test`，同名记录被去重 |

### 实验结论

Codex 使用 `SKILL.md` 的 `name` 作为展示名，但不去重。Claude Code 和 Cursor 更倾向使用目录名作为可见 ID。Copilot 按 `SKILL.md` 的 `name` 去重。

## 6. 实验四：目录名与 `name` 不一致时的链式引用

### 实验目的

验证当 skill 目录名与 `SKILL.md` frontmatter `name` 不一致时，CLI 是否仍能基于 skill name 触发并完成链式引用。

### 预期

如果 CLI 能稳定处理 frontmatter `name`，则应能从 `zz-fiji-hello` 继续调用 `zz-fiji-korean` 并输出韩文问候。

### 实验样板

| 目录名 | `SKILL.md` name | 作用 |
|---|---|---|
| `zz-fiji-hello-9asd9213jaskdj198ajsd` | `zz-fiji-hello` | 用户输入 `nice to meet you` 时，要求使用 `zz-fiji-korean` 问好 |
| `zz-fiji-korean-asjdasjdasjdasjdsaljdlqwejdqwj12` | `zz-fiji-korean` | 要求用韩文回复 `안녕하세요` |

### 实验步骤

1. 创建上述两个 skill。
2. 通过四个 CLI 的专属 skills 目录 symlink 加载。
3. 启动新 session。
4. 输入 `nice to meet you`。
5. 检查最终输出和 event 中的 skill 调用或文件读取。

### 判定标准

最终输出 `안녕하세요`，且 event 显示 skill 调用或 `SKILL.md` 读取，判定为成功。

### 实验结果

| CLI | 是否输出 `안녕하세요` | 事件证据 | 结论 |
|---|---:|---|---|
| Codex | 是 | stream 中读取两个长目录名下的 `SKILL.md` | 能根据初始化可见信息定位长目录并完成链式读取 |
| Claude Code | 是 | `system init` 包含两个长目录名；`Skill` 调用参数也是长目录名 | 能完成链式行为，但可调用 ID 以目录名为准 |
| Cursor | 否 | stream 中无 skill/tool 调用事件，最终为普通英文问候 | 未自动触发 skill |
| Copilot | 是 | `session.skills_loaded` 和后续 `skill` tool 参数均使用 frontmatter `name` | 明确支持按 `name` 触发和链式引用 |

### 实验结论

目录名与 `name` 不一致时，Cursor 在该样板下未触发。为跨 CLI 兼容，应避免目录名与 `SKILL.md` 的 `name` 不一致。

## 7. 实验五：目录名与 `name` 一致时的链式引用

### 实验目的

验证实验四中 Cursor 未触发是否与“目录名和 `name` 不一致”有关。

### 预期

目录名与 `name` 保持一致后，四个 CLI 均应完成链式引用。

### 实验样板

| 目录名 | `SKILL.md` name | 作用 |
|---|---|---|
| `zz-fiji-hello` | `zz-fiji-hello` | 用户输入 `nice to meet you` 时，要求使用 `zz-fiji-korean` 问好 |
| `zz-fiji-korean` | `zz-fiji-korean` | 要求用韩文回复 `안녕하세요` |

### 实验步骤

1. 保留实验四的 skill 内容和触发规则。
2. 去掉目录名中的随机后缀，使目录名与 `name` 一致。
3. 分别启动四个 CLI 新 session。
4. 输入 `nice to meet you`。
5. 检查最终输出和 event。

### 判定标准

最终输出 `안녕하세요`，且 event 显示 skill 调用或读取，判定为成功。

### 实验结果

| CLI | 是否输出 `안녕하세요` | 事件证据 | 与实验四相比 |
|---|---:|---|---|
| Codex | 是 | 读取 `zz-fiji-hello/SKILL.md` 与 `zz-fiji-korean/SKILL.md` | 行为一致 |
| Claude Code | 是 | `system init` 和后续 `Skill` 调用均使用一致名称 | 从长目录名调用变为按 name 调用 |
| Cursor | 是 | 读取 `.cursor/skills/zz-fiji-hello/SKILL.md` 与 `.cursor/skills/zz-fiji-korean/SKILL.md` | 从未触发变为成功触发 |
| Copilot | 是 | `session.skills_loaded` 和后续 `skill` tool 参数均为两个 name | 行为一致 |

### 实验结论

目录名与 `SKILL.md` 的 `name` 保持一致后，四个 CLI 均能完成链式 skill 场景。Cursor 在实验四失败，主要风险因素是目录名与 `name` 不一致。

## 8. 实验六：每个 skill 独立软链接

### 实验目的

验证 project skills 目录本身为普通目录、每个 skill 子目录分别为 symlink 时，四个 CLI 是否能正确加载。

### 预期

四个 CLI 均应加载 `bug-fix` 和 `say-hello` 两个 symlink skill。

### 实验样板

隔离测试项目：

```text
/Users/karmiy.hong/Documents/Projects-stage-2/FIJI2
```

外部 skill 目录：

| 外部目录 | `SKILL.md` name | marker |
|---|---|---|
| `/Users/karmiy.hong/.aidesktop/skills/bug-fix/1.2.0/` | `bug-fix` | `BUG_FIX_VERSION_1_2_0` |
| `/Users/karmiy.hong/.aidesktop/skills/say-hello/1.0.1/` | `say-hello` | `SAY_HELLO_VERSION_1_0_1` |

每个 CLI 的 project skills 目录下分别创建：

```text
.{cli}/skills/bug-fix   -> /Users/karmiy.hong/.aidesktop/skills/bug-fix/1.2.0/
.{cli}/skills/say-hello -> /Users/karmiy.hong/.aidesktop/skills/say-hello/1.0.1/
```

### 实验步骤

1. 每次仅保留当前 CLI 对应的 project skills 目录，避免交叉发现。
2. 启动新 session。
3. 要求只基于初始化可见 skill 列表回答 `bug-fix` 与 `say-hello` 是否存在。
4. 检查是否出现后续文件读取或目录搜索。

### 判定标准

首轮可见两个 skill，且未依赖后续文件读取，判定为加载成功。

### 实验结果

| CLI | 是否加载两个独立 symlink skill | 证据 |
|---|---:|---|
| Codex | 是 | 首轮列出 `bug-fix` / `say-hello`，包含两个 marker，无后续文件读取 |
| Claude Code | 是 | `system init` 的 `skills` / `slash_commands` 中包含两个 skill |
| Cursor | 是 | 首轮基于初始化 skill 列表列出两个 skill，路径为 `.cursor/skills/...` |
| Copilot | 是 | `session.skills_loaded` 中包含两个 skill |

### 实验结论

四个 CLI 均支持“project skills 目录为普通目录、每个 skill 子目录分别为 symlink”的形态。

## 9. 实验七：同一 ACP session 中 symlink 切版本

### 实验目的

验证外部 skill 已在 session 初始化阶段加载后，如果将 project skill symlink 从旧版本目录切换到新版本目录，后续同一 session 再次使用该 skill 时读取旧内容还是新内容；同时验证显式提醒和 `session/load` 的影响。

### 预期

如果 CLI 在后续调用时重新解析 symlink，应读取 `1.3.0`。如果 CLI 保留旧上下文或旧 realpath，应继续读取 `1.2.0`。

### 实验样板

第一轮 symlink：

```text
.{cli}/skills/bug-fix -> /Users/karmiy.hong/.aidesktop/skills/bug-fix/1.2.0/
```

第一轮期望输出：

```text
BUG_FIX_VERSION_1_2_0
```

随后将 symlink 切换为：

```text
.{cli}/skills/bug-fix -> /Users/karmiy.hong/.aidesktop/skills/bug-fix/1.3.0/
```

第二轮普通再次使用。第三轮显式提示 skill 已更新，要求重新加载当前 skill。第二轮和第三轮 prompt 均不包含 `BUG_FIX_VERSION_1_3_0`。

### 实验步骤

1. 通过 ACP 启动各 CLI adapter。
2. 第一组使用同一个 ACP live session 连续发送三次 `session/prompt`。
3. 第二组第一轮后关闭 ACP adapter，切换 symlink，重新启动 adapter。
4. 使用同一个 `sessionId` 执行 `session/load`。
5. 继续发送普通再次使用和提醒已更新两轮 `session/prompt`。
6. 检查 ACP 原始 JSON-RPC 事件、`available_commands_update`、tool 调用和返回文本。

### 判定标准

以实际返回 marker 为主要结果；以读取路径、tool 调用、`available_commands_update` 作为辅助证据。

### 实验结果

| CLI         | 第一轮   | Live 普通再次使用 | Live 提醒已更新后 | `session/load` 普通再次使用 | `session/load` 提醒已更新后 | 事件证据                                                                           |
| ----------- | ----- | ----------- | ----------- | --------------------- | --------------------- | ------------------------------------------------------------------------------ |
| Codex       | `1.2` | `1.2`       | `1.2`       | `1.2`                 | `1.2`                 | `session/load` 后 `available_commands_update` 已变为 `1.3`，但后续仍读取 `1.2.0/SKILL.md` |
| Claude Code | `1.2` | `1.2`       | `1.3`       | `1.2`                 | `1.3`                 | 普通再次使用受旧上下文影响；提醒后读到新版本                                                         |
| Cursor      | `1.2` | `1.2`       | `1.3`       | `1.2`                 | `1.3`                 | `session/load` replay 中仍可见旧 skill body；提醒后重新读取当前 skill                         |
| Copilot     | `1.2` | `1.2`       | `1.3`       | `1.3`                 | `1.3`                 | `session/load` 后普通再次使用即可加载 `1.3`                                               |

### Codex 机制说明

Codex 的 `session/load` 后，ACP adapter 发出的 `available_commands_update` 已刷新到 `BUG_FIX_VERSION_1_3_0`，说明当前 discovery 能看到 symlink 已切到 `1.3.0`。但同一 session 继续执行时，event 仍显示读取旧真实路径：

```text
/Users/karmiy.hong/.aidesktop/skills/bug-fix/1.2.0/SKILL.md
```

因此，Codex 至少存在两层状态：

| 状态层 | 观察结果 |
|---|---|
| 当前可用命令列表 | 可以随 adapter 重新扫描刷新到 `1.3` |
| 已存在 session 的实际 skill source / history | 仍可能保留旧 skill body 或旧 realpath 绑定 |

`available_commands_update` 刷新不能证明已存在 session 的 skill 绑定被重建。对 Codex 来说，`session/load` 不是可靠的 skill registry refresh 机制。

### 实验结论

切换 symlink 目标不适合作为 Codex 同一 session 内更新 skill 的机制。即使断开 adapter 并 `session/load`，Codex 仍可能沿用旧 realpath。

### 原始日志

| 类型 | 日志 |
|---|---|
| Live session | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp7/` |
| `session/load` | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp7-load/` |

## 10. 实验八：稳定路径原地更新

### 实验目的

验证方案 A：保持 project skill symlink 的目标 realpath 不变，只原地替换稳定路径下的 `SKILL.md`，同一个 ACP live session 后续是否能读到新内容。

### 预期

如果旧问题主要来自“symlink 切换导致 realpath 绑定停留在旧版本目录”，则稳定路径不变时，Codex 不应再被固定到 `1.2.0` 版本目录。原地更新 `SKILL.md` 后，后续读取同一路径应得到 `1.3`。

### 实验样板

稳定 materialized path：

```text
/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp8-stable-materialized/bug-fix/SKILL.md
```

project skill symlink：

```text
.{cli}/skills/bug-fix -> /Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp8-stable-materialized/bug-fix/
```

第一轮 `SKILL.md` 内容要求输出：

```text
BUG_FIX_VERSION_1_2_0
```

第一轮后不改变 symlink，只原地替换同一个 `SKILL.md` 文件内容，要求输出：

```text
BUG_FIX_VERSION_1_3_0
```

### 实验步骤

1. 启动 ACP live session。
2. 第一轮要求使用 `bug-fix`，确认输出 `1.2`。
3. 原地替换稳定路径下的 `SKILL.md` 为 `1.3` 内容。
4. 第二轮在同一个 session 中普通再次使用 `bug-fix`，不提示具体 marker。
5. 第三轮在同一个 session 中明确要求读取稳定路径的当前 `SKILL.md` 后再使用 skill。
6. 检查三轮返回文本和 event 中是否读取稳定路径。

### 判定标准

第二轮或第三轮输出 `BUG_FIX_VERSION_1_3_0`，且未出现旧版本目录 realpath，说明稳定路径方案可避免实验七中的旧 realpath 绑定问题。

### 实验结果

| CLI | 第一轮 | 原地更新后普通再次使用 | 明确重读稳定路径后 | 事件证据 | 结论 |
|---|---|---|---|---|---|
| Codex | `1.2` | `1.3` | `1.3` | 读取稳定路径；未出现 `1.2.0/SKILL.md` 或 `1.3.0/SKILL.md` 版本目录 realpath | 方案 A 对 Codex 成立 |
| Claude Code | `1.2` | `1.3` | `1.3` | 更新后可见 `available_commands_update` 到 `1.3`，并读取稳定路径 | 成立 |
| Cursor | `1.2` | `1.2` | `1.3` | 普通再次使用仍受旧上下文影响；明确重读稳定路径后读取新内容 | 需配合显式重读提示 |
| Copilot | `1.2` | `1.3` | `1.3` | 更新后读取稳定路径并输出新版本 | 成立 |

### 实验结论

稳定路径原地更新可以解决 Codex 在实验七中卡旧 realpath 的问题。对 Codex、Claude Code、Copilot，原地替换 `SKILL.md` 后普通再次使用即可读到新版本；对 Cursor，普通再次使用仍可能受旧上下文影响，但明确要求读取稳定路径后可读到新版本。

### 工程建议

如需在同一产品会话内支持 skill 更新，优先采用以下机制：

1. project skill 目录中使用稳定 symlink，例如 `.codex/skills/bug-fix -> <stable-materialized>/bug-fix/`。
2. 版本更新时原地替换 `<stable-materialized>/bug-fix/SKILL.md`，不要把 symlink 从 `1.2.0` 切到 `1.3.0`。
3. daemon 检测 skill fingerprint 变化后，在下一轮 prompt 自动注入“该 skill 已更新，请重新读取稳定路径下的 `SKILL.md`”。
4. 对 Codex，不应依赖 `session/load` 刷新已使用过的 skill；稳定路径原地更新比切换版本目录 symlink 更可靠。

### 原始日志

| CLI | 日志文件 |
|---|---|
| Codex | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp8/codex-exp8-stable-2026-07-07T16-03-15-744Z.json` |
| Claude Code | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp8/claude-exp8-stable-2026-07-07T16-04-35-305Z.json` |
| Cursor | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp8/cursor-exp8-stable-2026-07-07T16-05-01-610Z.json` |
| Copilot | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp8/copilot-exp8-stable-2026-07-07T16-06-37-399Z.json` |

## 11. 实验九：workspace 中项目目录为 symlink

### 实验目的

验证当当前工作区本身不是项目源码目录、而是只包含一个 `FIJI` symlink 指向真实项目时，四个 CLI 是否能沿 symlink 进入真实项目；同时验证在 workspace 根目录增加 `AGENTS.md` 后，是否能继续正确识别真实项目。

### 预期

如果 CLI 不能处理该 workspace 形态，可能只看到当前目录下一个 `FIJI` 条目，甚至误判为“空目录”或“无源码”。如果 CLI 能处理 symlink，则应进入 `/Users/karmiy.hong/Documents/Projects-stage-2/Fiji` 并识别其前端 monorepo / 微前端架构。

### 实验样板

创建临时 workspace：

```text
/Users/karmiy.hong/.aidesktop/workspace-exp9-workspace-symlink-2026-07-08T00-20-00
```

workspace 内容：

```text
FIJI -> /Users/karmiy.hong/Documents/Projects-stage-2/Fiji
```

第二组实验额外增加 `AGENTS.md`：

```markdown
# Workspace Instructions

The FIJI directory in this workspace is a symbolic link to:

/Users/karmiy.hong/Documents/Projects-stage-2/Fiji

For architecture questions, treat FIJI as the project root. Follow the symlink and inspect the real project files before answering. Do not conclude the workspace is empty only because the current directory contains a symlink.
```

### 实验步骤

1. 创建 workspace，并在其中创建 `FIJI` symlink。
2. 第一组不创建 `AGENTS.md`，分别以 workspace 根目录作为 cwd 启动四个 CLI 的 ACP 新 session。
3. 向四个 CLI 发送同一问题：`请检查当前工作区，回答这个项目是什么架构。请先确认当前目录内容，再说明你是否能看到 FIJI 项目源码。`
4. 第二组创建上述 `AGENTS.md`，重新启动四个 CLI 的 ACP 新 session。
5. 重复同一问题。
6. 检查回答是否误判空目录、是否识别 symlink、是否进入真实 Fiji 项目、是否提到 `application/`、`packages/`、Yarn Workspaces、TypeScript、monorepo / 微前端等真实结构。

### 判定标准

回答明确说明 `FIJI` 是 symlink，且能识别真实 Fiji 项目为前端 monorepo / 微前端架构，判定为能读到真实项目。若回答当前 workspace 为空、无源码，或无法进入 `FIJI`，判定为失败。

### 实验结果

| CLI | 无 `AGENTS.md` 是否读到真实项目 | 有 `AGENTS.md` 是否读到真实项目 | 是否误判空目录 | 事件 / 回答证据 |
|---|---:|---:|---:|---|
| Codex | 是 | 是 | 否 | 两组均识别 `FIJI -> /Users/karmiy.hong/Documents/Projects-stage-2/Fiji`，并提到 `application/*`、`project/*`、`packages/*`、Yarn workspaces、TypeScript/React、monorepo |
| Claude Code | 是 | 是 | 否 | 两组均识别 symlink 与真实路径；有 AGENTS 组明确按 AGENTS 指引读取真实项目根目录 |
| Cursor | 是 | 是 | 否 | 两组均识别 symlink 与真实路径；有 AGENTS 组明确读取 `AGENTS.md` 并进入真实项目 |
| Copilot | 是 | 是 | 否 | 两组均识别 symlink 与真实路径，并描述 `application/`、`packages/`、TypeScript、Jest、monorepo / MFE |

### AGENTS.md 影响

本样板中，无 `AGENTS.md` 时四个 CLI 已经都能沿 symlink 进入真实项目，因此该实验不能证明 `AGENTS.md` 是必要条件。增加 `AGENTS.md` 后，四个 CLI 仍均能正确读取真实项目。

AGENTS 读取证据存在差异：

| CLI | AGENTS 读取证据 |
|---|---|
| Codex | 回答提到“按工作区说明”，但未观察到单独读取 `AGENTS.md` 的文件读取事件；可能是初始化上下文注入 |
| Claude Code | 有 AGENTS 组明确表示按 `AGENTS.md` 指引读取真实项目 |
| Cursor | 有 AGENTS 组明确列出 `AGENTS.md` 并按其说明处理 symlink |
| Copilot | 有 AGENTS 组能看到 `AGENTS.md` 文件存在，但本次事件不足以证明其显式消费了 AGENTS 内容 |

### 无 AGENTS 多轮复测

由于存在历史反馈称“无 AGENTS.md 时，某些 CLI 曾把项目判断为空目录”，对无 `AGENTS.md` 场景追加 3 轮复测。每轮均创建新的 workspace、新的 `FIJI` symlink，并为四个 CLI 启动新的 ACP session。

复测范围：

| 轮次 | workspace | AGENTS.md |
|---|---|---|
| 原始轮次 | `/Users/karmiy.hong/.aidesktop/workspace-exp9-workspace-symlink-2026-07-08T00-20-00` | 无 |
| 复测 1 | `/Users/karmiy.hong/.aidesktop/workspace-exp9-repeat-noagents-r1-2026-07-08T10-02-52` | 无 |
| 复测 2 | `/Users/karmiy.hong/.aidesktop/workspace-exp9-repeat-noagents-r2-2026-07-08T10-06-49` | 无 |
| 复测 3 | `/Users/karmiy.hong/.aidesktop/workspace-exp9-repeat-noagents-r3-2026-07-08T10-10-40` | 无 |

汇总结果：

| CLI | 无 AGENTS 总轮次 | 读到真实 Fiji 项目 | 真正误判为空目录 | 备注 |
|---|---:|---:|---:|---|
| Codex | 4 | 4 | 0 | 复测 3 轮中均先指出 workspace 根目录只有一个 symlink / 空壳工作区，但随后继续跟随 symlink 并正确识别 Fiji 架构；不属于最终误判为空目录 |
| Claude Code | 4 | 4 | 0 | 每轮均明确说明 `FIJI` symlink 指向真实路径，并能访问源码 |
| Cursor | 4 | 4 | 0 | 每轮均明确说明可通过 symlink 访问完整 Fiji 源码 |
| Copilot | 4 | 4 | 0 | 每轮均说明能看到 Fiji 源码，并给出 monorepo / 微前端架构判断 |

复测结论：

- 本机当前环境下，4 轮无 `AGENTS.md` 测试均未复现“CLI 把项目最终判断为空目录”的问题。
- Codex 会更谨慎地描述“当前 workspace 根目录只有一个 symlink / 不是 git 仓库 / 像空壳工作区”，但它随后会继续检查 `FIJI/` symlink 目标并识别真实项目。
- 如果业务上希望避免模型停留在“当前目录是空壳”这个中间判断，父目录 `AGENTS.md` 仍有价值：它能显式告诉 CLI 把 `FIJI` 当作项目根目录并跟随 symlink。

### 实验结论

当前测试条件下，四个 CLI 都能处理“workspace 根目录只包含项目 symlink”的形态，并能沿 `FIJI` symlink 进入真实 Fiji 项目。`AGENTS.md` 可作为额外保险和人类可读说明，但在本样板中不是让 CLI 读到项目源码的必要条件。

### 原始日志

| 类型 | 日志 |
|---|---|
| 汇总 | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp9/summary-exp9-workspace-symlink-2026-07-08T00-20-00.json` |
| 无 AGENTS 复测 1 | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp9/summary-exp9-repeat-noagents-r1-2026-07-08T10-02-52.json` |
| 无 AGENTS 复测 2 | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp9/summary-exp9-repeat-noagents-r2-2026-07-08T10-06-49.json` |
| 无 AGENTS 复测 3 | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp9/summary-exp9-repeat-noagents-r3-2026-07-08T10-10-40.json` |
| 原始事件 | `/Users/karmiy.hong/Documents/Codex/2026-07-07/bug-codex-claude-skill-skill/work/acp-exp9/` |

## 12. 最终建议

1. 跨 CLI 兼容时，不使用 `.agents/skills` 作为唯一入口；应生成四个专属入口：`.codex/skills`、`.claude/skills`、`.cursor/skills`、`.github/skills`。
2. skill 目录名与 `SKILL.md` 的 `name` 保持一致，避免 Cursor、Claude Code、Copilot 在调用 ID 和展示 ID 上出现差异。
3. 避免多个目录使用相同 `SKILL.md name`；Copilot 会去重，Codex 不去重，跨 CLI 行为不一致。
4. 如果需要版本化 skill，不建议将 project symlink 直接指向 `1.2.0`、`1.3.0` 这类版本目录再切换。
5. 推荐使用稳定 materialized path，并在版本变化时原地替换 `SKILL.md` 内容。
6. daemon 应维护 skill fingerprint。fingerprint 变化后，对同一 UI conversation 的下一轮 prompt 注入重新读取稳定路径的要求。
7. 对 Codex，`session/load` 只能恢复会话，不应被当作刷新已使用 skill 的机制。
8. 对 workspace 级项目软链接，可保留父目录 `AGENTS.md` 作为说明和保险，但本次实验显示四个 CLI 即使没有 AGENTS 也能沿项目 symlink 进入真实源码。

## 13. 证据边界

本报告未抓取底层 provider API request payload，因此不声称逐字证明最终 prompt 中包含哪些 skill。报告结论基于 CLI 初始化事件、ACP stream event、tool 调用、文件读取路径、模型输出和无后续搜索事件链。

测试完成后，Fiji / FIJI2 项目目录均已恢复为实验前预期状态。
