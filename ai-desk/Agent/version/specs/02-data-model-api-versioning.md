# 02 数据模型、API 与版本规则

更新时间：2026-07-07

## 数据模型

在 `agents` 表新增真实字段：

```text
version varchar(20) not null default '1.0.0'
```

Migration 规则：

- migration 只放在根目录 `db_migrations/versions/`。
- 使用 additive migration。
- 已有 rows 自动获得 `1.0.0`。
- revision id 长度遵守仓库 Alembic 限制。
- 不要在 `backend/alembic/versions/` 或 `backend_gleam/db_migrations/versions/` 新增 mirror migration。

Model/schema 更新：

- `backend/app/models/agent.py`: add `Agent.version`.
- `backend/app/schemas/agent.py`: add `version: str` to `AgentRead`.
- `frontend/src/types/agent.ts`: add `version: string` to `Agent`.
- Gleam Agent catalog response 同步包含 `version`。

## Version 格式

格式：

```text
<major>.<minor>.<patch>
```

默认值：

```text
1.0.0
```

本期 Agent 只递增 patch。

非法或缺失 version 在后端 helper 中按 `1.0.0` 处理后再递增。这只是防御性逻辑；DB 字段正常应为 non-null。

如果业务校验、权限校验、DB 写入或其他 update 流程失败，必须保持当前 DB version 不变。不能因为 helper fallback 把已有 version 回退成 `1.0.0`。

## 后端 Helper

参考 Skill / Skill Set 的实现风格，但 Agent 只需要 default + patch bump。Python backend 和 Gleam backend 必须同步实现同等规则。

Python helper 建议：

```python
DEFAULT_AGENT_VERSION = "1.0.0"

def get_agent_version(value: object) -> str:
    ...

def bump_agent_patch(value: object) -> str:
    ...
```

Gleam helper 建议放在 Agent catalog feature 的 pure policy/helper 中，或当前迁移阶段最接近的 Agent catalog policy module 中：

```gleam
pub const default_agent_version = "1.0.0"

pub fn get_agent_version(value: String) -> String {
  ...
}

pub fn bump_agent_patch(value: String) -> String {
  ...
}
```

如果 Gleam 当前 update path 仍用 SQL mutation plan，也可以提供等价 SQL expression/helper，但规则必须集中，不能在多个 SQL 字符串里手写不同版本解析逻辑。

规则：

- `get_agent_version(None) == "1.0.0"`
- `get_agent_version("bad") == "1.0.0"`
- `get_agent_version("2.3.4") == "2.3.4"`
- `bump_agent_patch(None) == "1.0.1"`
- `bump_agent_patch("bad") == "1.0.1"`
- `bump_agent_patch("1.0.31") == "1.0.32"`

helper 必须集中，避免在 Python route/service 或 Gleam route/SQL 中散落字符串解析逻辑。两边测试用例应覆盖同一组输入输出。

## API Contract

### `POST /api/v1/agents`

创建成功返回：

```json
{
  "version": "1.0.0"
}
```

request body 不需要 `version` 字段。即使客户端误传，后端也不信任、不使用。

### `PATCH /api/v1/agents/{agentId}`

成功保存可编辑字段时：

```text
agent.version = bump_agent_patch(agent.version)
```

response 返回递增后的 version。

版本递增必须和 Agent 其他字段更新在同一个成功事务里提交。任何失败路径都不更新 version。

如果 payload 没有任何可识别的可编辑字段，建议返回 `400`，或返回 unchanged 且不 bump。空 no-op 请求不应产生版本递增。

### 其他 Agent endpoint

list/detail/admin list 都应在 `AgentRead` 中包含 `version`。

install、uninstall、delete、usage endpoint 保持当前 version，不递增。

## OpenAPI

如果 canonical OpenAPI 中包含 Agents schema/endpoint，需要同步更新 Agent response schema：

- `AgentRead.version` is required.
- Agent create/update request schema 不要求 `version`。

如果 Agents 当前只通过 Pydantic schema 和 frontend types 对齐 contract，则在实现说明中记录该事实并保持两边同步。
