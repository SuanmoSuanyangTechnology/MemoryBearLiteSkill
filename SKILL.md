---
name: memory-skills
description: 通过 MemorySkills 完成邮箱验证码免密注册/登录，领取记忆 API Key（sk-mem-），配置到 AI 客户端。请求经 Skills 校验记忆 Key 后，用服务端持有的空间 API Key（sk-service-）代理调用 MemoryBear REST 接口读写记忆。
---

# MemorySkills

通过 MemorySkills Portal 完成身份验证，领取属于你的记忆 API Key（`sk-mem-`），用于 MCP 客户端 / SDK / Skill 访问 MemoryBear 记忆服务。

Skills 服务端持有空间 API Key（`sk-service-`，绝不下发），校验用户的记忆 Key 后代理调用 MemoryBear 的 REST 写/读接口。用户的 `sk-mem-` Key 不会直连 MemoryBear。

## 固定约束

- **已确认的部署环境**：当前默认 Portal 为 `https://memoryskills.redbearai.com`（HTTPS 生产环境，已由用户确认）。若用户在对话中提供其他 Portal 地址，以其为准。
- **Portal 基础 URL** 必须由用户在对话中显式提供，或由 Skill 宿主客户端以参数方式注入。未获得前不要发起任何请求，不要猜测 `localhost` / `<lan-ip>` 等默认值。
- 拿到 Portal 基础 URL 后，仍然要调 `/api/runtime-config` 读取 `data.skills_base_url` 作为后续所有拼接的权威值（用户提供的可能带尾斜杠、协议大小写不规范，或指向反向代理入口）。当前环境该值为 `https://memoryskills.redbearai.com`。
- 登录 + 凭证页面：`/#/login`（HashRouter，邮箱 + 验证码免密登录，登录成功后同页面切换为凭证卡视图，展示可复制的 API Key）
- 登录成功后凭证卡视图提供"查看记忆图谱"按钮，点击进入记忆查看页（知识图谱 + 记忆活动记录：写入记忆 / 引擎动态），即"重新访问服务地址查看自己的记忆信息"的入口；跳转由页面自动完成，无需 Skill 拼接或传递 `end_user_id`
- 运行时配置接口：`/api/runtime-config`
- MCP 路径：`/v1/mcp/memory/`（**带尾斜杠**，缺失会触发 307 重定向）
- MCP 工具：`read_memory`、`write_memory`

不要接收、生成或传递 `workspace_id`、`end_user_id` 或 `other_id`。用户身份已经绑定在 Session Cookie 或 Key 上。

## 安全边界

当前已确认的部署环境 `https://memoryskills.redbearai.com` 使用 HTTPS，可用于生产。若用户提供的是 HTTP / 局域网地址，只能在用户确认的受信任局域网开发环境中使用；若地址不是上述 Portal 地址、网络不可信或准备投入生产，停止并说明必须改用 HTTPS。

始终遵守：

- 把 API Key 当作密码处理。
- 不在回复、日志、命令输出或错误信息中回显完整 Key。
- 不把 Key 写入项目目录、代码仓库、规则文件、测试文件或历史记录。
- 优先使用客户端的 Secret、Input 或环境变量引用。
- 客户端不支持安全引用时，先征得用户同意，再把 Key 写入用户级 MCP 配置。
- 不自动读取剪贴板。只有用户明确确认"已复制并允许读取剪贴板"后才能读取一次。
- 不保存邮箱验证码。
- 不把验证码、API Key 或其他认证信息写入记忆。

## 接入流程

### 1. 检查 Portal

当前默认 Portal 地址为 `https://memoryskills.redbearai.com`（若用户提供了其他地址，优先使用用户提供的值）。请求：

```text
GET <portal>/health/live
GET <portal>/api/runtime-config
```

要求：

- `/health/live` 返回 HTTP 200。
- `/api/runtime-config` 返回统一响应，读取 `data.skills_base_url`。
- 删除 Base URL 末尾的 `/`，再拼接 `/v1/mcp/memory/`（尾斜杠必须保留）得到 MCP URL。
- 不从运行配置中寻找 Workspace、End User 或空间 Key。

Portal 不可访问时停止。不要生成假账号、假 Key、假数据或切换到 Mock。

### 2. 引导用户在浏览器中完成 Passwordless 登录并领取 API Key

把 `<skills_base_url>/#/login` 提供给用户（有浏览器打开能力时优先直接打开）。当前环境即 `https://memoryskills.redbearai.com/#/login`。让用户在浏览器中自主完成整个流程：

1. 输入邮箱。
2. 接收并输入邮箱验证码（生产走邮件；开发模式下页面上会自行显示）。
3. 提交后新用户自动注册，已有用户自动登录。
4. **登录成功后同一页面切换为"凭证卡"视图**，展示 `email` / `api_key` / `end_user_id` 三行，`api_key` 一行右侧带独立的 **复制** 按钮；页面底部另有一个 **复制全部凭证** 按钮，会一次性复制三行文本。
5. **首选**：让用户点击 `api_key` 行右侧的 **复制** 按钮，剪贴板拿到的是纯 Key（`sk-mem-...`）。
6. **降级**：如果用户误点了底部的 **复制全部凭证**，剪贴板内容形如：
   ```text
   email: user@example.com
   api_key: sk-mem-xxxxxxxx...
   end_user_id: 710a6d41-8391-49d6-a48e-...
   ```
   此时从 `api_key:` 前缀那一行提取值，去掉前后空白即可；**不要**把 `email` 或 `end_user_id` 一起写进 MCP 配置。
7. **查看记忆信息**：凭证卡视图提供"查看记忆图谱"按钮，点击进入记忆查看页，可浏览知识图谱与记忆活动记录（写入记忆 / 引擎动态）。用户之后随时重新访问 `<skills_base_url>/#/login`，登录后即可查看自己已写入的记忆信息；跳转由页面自动完成，无需 Skill 拼接或传递 `end_user_id`。

整个流程不需要密码，也不需要在 Skills 侧新建"设置页"——凭证卡本身就是领 Key 的地方。同一账号无论登录多少次，返回的都是同一把 Key，不会滚动生成新 Key。

获取 Key 时按以下顺序选择（**AI 尽量不看到 Key 明文**）：

1. 客户端支持 Secret 或敏感 Input：让用户直接把 Key 粘贴到客户端的安全输入框，AI 不接触明文。
2. Agent 能读取系统剪贴板：先取得用户明确许可，再读取一次；使用完立即从工作变量中清除，不在回复中输出 Key。（参见"安全边界"关于剪贴板的约束）
3. 以上均不可用：让用户把 Key 粘贴到聊天，但事先提醒聊天记录可能长期保存该值。

**约束**：
- 不要向用户索取验证码；不要尝试用 Shell / curl 替用户走注册；不要读取 HttpOnly Session Cookie。
- 用户重复访问 `/#/login` 时，只需再走一次邮箱验证码即可再次看到同一把 Key，不需要额外的"设置页"。**同一账号任意次数登录返回的都是同一把 Key，不会自动滚动生成新 Key**——所以 AI 不必担心"再登录一次会不会让原来配置里的 Key 失效"。
- **Key 轮换**：如果用户怀疑 Key 泄露或想主动换一把，当前 MemorySkills 没有面向用户的自助轮换能力，必须联系管理员在服务端处理。AI 不要尝试重复注册或调用未记录的接口去"生成新 Key"。

### 3. 安装 MCP

先识别当前 AI 客户端及其用户级 MCP 配置位置。优先使用客户端自带的 MCP 管理命令或设置界面；否则读取并合并用户级 JSON 配置。

不要优先使用项目级配置。只有用户明确要求项目级配置并接受 Key 泄露风险时才允许使用。

目标连接的逻辑结构是：

```json
{
  "memorybear": {
    "type": "http",
    "url": "<runtime-config 返回的 skills_base_url>/v1/mcp/memory/",
    "headers": {
      "Authorization": "Bearer <End User API Key>"
    }
  }
}
```

当前环境 URL 为 `https://memoryskills.redbearai.com/v1/mcp/memory/`（**带尾斜杠**）；但仍以 `/api/runtime-config` 返回的 `skills_base_url` 为权威拼接基准，避免协议/尾斜杠写错。

根据客户端现有格式调整顶层键名和字段：

- 使用 `mcpServers` 的客户端（Claude Desktop / Cline / Continue 等）：把上面的 `memorybear` 放入 `mcpServers`。
- VS Code：使用 `servers`，并保留 `"type": "http"`。
- Cursor：字段名是 `transport` 而非 `type`；同一逻辑结构写成 `"transport": "http"`。
- 若客户端不识别 `"type": "http"`，可依次尝试 `"streamable-http"` 或该客户端专用字段；参照该客户端当前文档，不要猜测格式。
- 使用其他结构的客户端：先读取现有配置或查阅该客户端当前文档，不要猜测格式。

不要添加：

```text
X-End-User-Other-Id
other_id
end_user_id
workspace_id
```

凭证存储按以下优先级处理：

1. 客户端 Secret/Input。
2. 客户端明确支持的环境变量引用。
3. 用户明确同意后的用户级明文 Header。

不要自动修改 `.zshrc`、`.bashrc`、PowerShell Profile 或系统环境。GUI 客户端不一定继承 Shell 环境变量。

客户端仅支持 stdio 时，可以使用 `mcp-remote` 桥接，但必须：

- 使用经过确认的固定版本，例如 `mcp-remote@<verified-version>`。
- 先向用户说明会安装 Node 包并获得同意。
- 只传 `Authorization` Header。
- 不使用未固定版本的 `npx -y mcp-remote`。

### 4. 安全合并配置

修改配置前：

1. 读取原文件。
2. 确认目标是用户级配置。
3. 检查是否已经存在 `memorybear`。
4. 已存在时先询问用户是否更新。

修改时：

1. 保留全部现有 Server 和其他字段。
2. 只新增或更新 `memorybear`。
3. 先建立同目录备份。
4. 使用原子写入，避免中途留下损坏 JSON。
5. 限制配置文件权限，避免其他本机用户读取。

修改后：

1. 重新读取并解析 JSON。
2. 确认原有 Server 仍然存在。
3. 确认没有 `X-End-User-Other-Id`。
4. 检查项目目录和 Git Diff，确认没有写入 Key。
5. 不打印包含完整 Header 的配置。

### 5. 验证接入

让用户重新加载或重启当前 AI 客户端，然后：

1. 确认 MCP Server `memorybear` 已连接。
2. 确认工具列表包含 `read_memory` 和 `write_memory`。
3. 调用 `read_memory(message="ping")` 或 `read_memory(message="用户偏好")` 做一次无副作用查询——**注意 `message` 不能为空**，空串会被服务端拒绝。新用户或没有相关记忆的账号会返回 `answer: ""`，这**不是**故障。
4. 只有用户明确同意时，才写入一条真实且有价值的记忆用于验证。
5. 可选验证"查看记忆信息"：请用户重新访问 `<skills_base_url>/#/login` 登录，通过凭证卡的"查看记忆图谱"按钮进入记忆查看页，确认写入的记忆出现在活动记录/知识图谱中（写入为异步处理，可能需稍候）。

不要为了测试写入随机个人信息。当前没有通用删除工具，测试垃圾可能长期保留。

验证失败时，只报告脱敏信息，包括：

- Portal 是否可访问。
- MCP URL 的域名和路径。
- 客户端配置类型。
- HTTP 状态码或脱敏错误。

不要报告完整 Key 或完整 Authorization Header。

## 读写记忆

### 读取

当用户的问题依赖历史偏好、此前决策、长期上下文或用户明确要求回忆时，调用：

```text
read_memory(message="<与当前问题直接相关的自然语言查询>", search_switch="express")
```

参数：
- `message`（必填）：查询文本，不能为空。
- `search_switch`（可选，默认 `"express"`）：检索深度档位。
  - `"express"`：快速召回，适合大多数上下文查询（默认）。
  - `"deep"`：更完整地扫描历史记忆，延迟高一些，适合"用户明确要求详细回忆"的场景。
  - `"research"`：最深档，延迟最高；仅在用户明确要求做研究性回顾时使用。
  - 传其他值会被工具直接拒绝（`ToolError`）。

不要机械地在每轮对话前读取。查询应聚焦当前任务，避免检索无关隐私。

### 写入

当用户明确要求记住，或提供适合跨会话保存的稳定偏好、长期决策和持续事项时，调用：

```text
write_memory(message="<简洁、完整、可独立理解的记忆>")
```

参数与约束：
- `message`（必填）：单条记忆文本；**1–8000 字符**，超长会被工具直接拒绝（`ToolError`）。需要长文时先摘要或拆条写入。
- 每把 Key 每分钟最多 **30 次** `write_memory` 调用；触发限流时会返回错误，见"故障处理"。

写入前：

- 不把推测当作事实。
- 不创造用户未提供的个人资料。
- 不写入密码、验证码、Token、API Key 或其他 Secret。
- 对敏感个人信息先询问用户是否确实希望长期保存。
- **不要为了"补齐历史"批量导入过去对话**——限流之外，也会污染用户长期记忆库。

`write_memory` 成功只表示异步任务已经提交，不表示记忆已经完成处理。不要承诺立即可检索；需要确认时，等待后台处理后再读取。

## 故障处理

- Portal 无法访问：停止并让用户检查局域网地址、服务和防火墙。
- 注册验证码未收到：让用户检查邮件服务和垃圾邮件（生产模式），或直接看登录页上的验证码提示（开发模式）；不伪造验证码。
- `POST /api/auth/send-code` 直接返 429：邮箱或出口 IP 触发发码限流（单邮箱 5 次/小时；同 IP 同邮箱组合 3 次/小时；单 IP 全局 100 次/小时；IP 请求量超过阈值后升级为人机验证）。让用户等待窗口期过后重试，或换出口 IP；不要让用户反复点击"发送验证码"。
- **`write_memory` / `POST /api/v1/memory/write` 返回 429**：写入超频（每 Key 30 次/分钟）。停止本轮写入、告知用户稍后重试；**不要**做自动重试或自适应降速循环——那会持续踩红线。
- **`ToolError: message 长度不能超过 8000 字符`**：AI 写入内容太长；先摘要或拆成多条 ≤ 8000 字符的记忆分别写入。
- **`ToolError: search_switch 参数无效`**：只接受 `express` / `deep` / `research` 三个值；不要传其他自造词。
- 凭证卡未出现（点了登录但页面无反应）：让用户刷新 `/#/login` 并重新走一次验证码流程。
- MCP 返回 401（含 `WWW-Authenticate: Bearer`）：Header 未带 `Authorization` 或 Key 已失效；让用户回到 `/#/login` 重新登录一次，凭证卡里复制同一把 Key 并更新客户端配置。
- 登录提交返回 403 `CSRF_ORIGIN_INVALID`：浏览器 Origin 与服务器 `PORTAL_PUBLIC_ORIGIN` 不匹配。常见原因：前端走 HTTPS 而白名单配了 HTTP（如 `https://memoryskills.redbearai.com` vs `http://...`），或反代改写/丢弃 Origin。让管理员核对环境变量协议与浏览器地址一致。
- Key 已禁用：当前重复领取不会自动轮换，停止并让管理员恢复或更换 Key。
- 工具未出现：检查配置结构、MCP URL、客户端是否支持远程 HTTP，以及是否已经重新加载。
- 写入后立即读不到：按异步处理看待，稍后重试，不重复批量写入。

## 完成标准

只有同时满足以下条件才报告接入完成：

- 用户已在浏览器 `/#/login` 页面通过 Passwordless 完成注册或登录。
- 用户已在同一页面的凭证卡中主动点击 **复制** 或将 API Key 交给 Agent。
- Key 只存在于用户批准的安全存储或用户级配置。
- MCP URL 来自 `/api/runtime-config` 的 `skills_base_url` 拼接 `/v1/mcp/memory/`。
- `memorybear` MCP 已连接成功。
- `read_memory` 和 `write_memory` 均出现在工具列表。
- 项目文件和 Git Diff 中不存在 Key。
