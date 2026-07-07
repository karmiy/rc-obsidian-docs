# Standalone Skill MCP Prompt 方案图

更新时间：2026-07-06

## 运行链路

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant FE as Frontend<br/>SlashPromptComposer
    participant RT as Runtime AI
    participant MCP as Backend MCP
    participant AR as Archive Endpoint
    participant DB as DB
    participant FS as Runtime Workspace

    U->>FE: 选择 /bug-fix
    FE->>FE: 保存 marker<br/>[[vibeSkill|aidesk|bug-fix|id=skillId]]
    FE->>RT: Expanded prompt<br/>AI Desk @skill "bug-fix"<br/>get_platform_entities(id=skillId, kind=skill)
    RT->>MCP: call get_platform_entities<br/>{ id: skillId, kind: "skill" }
    MCP->>DB: 查询 Skill + files + version<br/>校验可见性/权限
    DB-->>MCP: Skill bundle metadata
    MCP-->>RT: metadata + materialization + agentInstructions
    RT->>FS: 检查 materialization.targetDir 是否存在
    alt targetDir 已存在
        FS-->>RT: 复用本地缓存
    else targetDir 不存在
        RT->>AR: GET materialization.archiveUrl
        AR->>DB: 验 token 后重新查询 DB/权限/version
        DB-->>AR: Skill files
        AR-->>RT: zip archive
        RT->>FS: 解压到 targetDir
    end
    RT->>FS: 读取 materialization.entryFile<br/>SKILL.md
    RT->>RT: 严格遵循 response.agentInstructions
```

## Prompt 与兼容策略

```mermaid
flowchart TD
    A["用户选择 standalone Skill<br/>/bug-fix"] --> B["Internal marker<br/>[[vibeSkill|aidesk|bug-fix|id=&lt;skillId&gt;]]"]
    B --> C["新 canonical prompt<br/>[AI Desk @skill &quot;bug-fix&quot;: call MCP tool get_platform_entities(id='&lt;skillId&gt;', kind='skill') and strictly follow response.agentInstructions]"]
    C --> D["Runtime MCP<br/>get_platform_entities(kind='skill')"]
    D --> E["返回 materialization + agentInstructions"]

    L["历史 workflow prompt<br/>get_platform_skill(skill_id=..., display_slash=...)"] --> R["Reverse parse"]
    R --> B
    L --> G["Legacy runtime fallback<br/>get_platform_skill 保留可用"]

    B --> H{"用户是否编辑 prompt?"}
    H -- "是" --> C
    H -- "否" --> L

    classDef newPath fill:#e7f5ff,stroke:#1c7ed6,color:#0b1f33;
    classDef legacy fill:#fff5f5,stroke:#e03131,color:#3b0a0a;
    classDef neutral fill:#f8f9fa,stroke:#868e96,color:#212529;

    class A,B,C,D,E newPath;
    class L,G legacy;
    class R,H neutral;
```

## 关键约定

- 新 prompt 只把 `id` 和 `kind` 作为 MCP 参数传给后端。
- `AI Desk @skill "bug-fix"` 中的 display name 只用于人读和前端反解，不参与后端 fallback。
- 独立 Skill 反解后仍使用 `vibeSkill` marker，不新增 `promptMention|skill`。
- 不做全局 save-time canonicalize；旧 workflow prompt 只有在用户编辑该 prompt 后才机会式迁移为新格式。
- `get_platform_skill` 不删除，继续服务历史 prompt。
