<div align="center">

<img src="assets/cover.png" alt="MemoryLiteSkills 封面" width="820" />

# MemoryLiteSkills

**给你的 AI 客户端接上一份可以跨会话记住你的长期记忆。**

免密登录领取记忆钥匙 · 服务端代理调用 · 一次接入随处可用

</div>

---

## 这是什么

MemorySkills 是一个帮助你把 **MemoryBear 记忆服务**接入任意 AI 客户端（Claude Desktop、Cursor、VS Code、Cline 等）的接入向导（Skill）。

它解决一件事：让你的 AI 助手拥有**跨会话的长期记忆**——记住你的偏好、过往决策和长期事项，而不用每次从头解释。

整个接入过程不需要密码、不需要手动申请，只要在浏览器里用邮箱验证码登录一次，就能领到属于你的记忆钥匙（API Key），然后由 Skill 帮你安全地写进客户端配置。

---

## 如何使用：把 SKILL.md 交给你的 Agent

这份 Skill 的核心用法是**让 AI Agent 代你完成整个接入**。你几乎不用动手，只需在浏览器里点几下。

把本目录下的 [`SKILL.md`](./SKILL.md) 提供给你的 AI Agent（作为 Skill 加载，或直接把内容贴进对话），然后对它说一句类似：

> 「按这份 Skill 帮我接入 MemoryBear 记忆服务。」

Agent 会依据 SKILL.md 自动执行以下流程：

1. **检查服务** — 探测 Portal 的 `/health/live` 与 `/api/runtime-config`，拿到权威的 MCP 地址。
2. **引导你领 Key** — 帮你**打开注册/登录网址** `<portal>/#/login`，你在浏览器里用邮箱验证码免密登录，从凭证卡复制 `sk-mem-` 钥匙（Agent 尽量不接触明文）。
3. **自动配置 MCP** — Agent 识别你当前的 AI 客户端，把 `memorybear` 这一段**自动写入用户级 MCP 配置**（先备份、原子写入、只新增不破坏已有配置）。
4. **验证接入** — 重载客户端后确认 `memorybear` 已连接、`read_memory` / `write_memory` 工具已就位，并做一次无副作用的 `read_memory` 查询。

你需要亲自做的只有两件事：**在浏览器完成邮箱验证码登录**、**把钥匙交给 Agent 或粘进客户端的安全输入框**。其余的检查、写配置、验证都由 Agent 依 SKILL.md 完成。

> 为什么要人来登录？因为领 Key 走的是浏览器里的免密验证，Agent 不会替你走验证码、也不读你的登录 Cookie——这是刻意的安全边界（详见下文「安全边界」）。

---

## 核心思路：你的钥匙不直接连数据库，中间隔着一道关卡

<div align="center">
<img src="assets/architecture.png" alt="三道门：你和客户端 → MemorySkills 关卡 → MemoryBear 记忆库" width="820" />
</div>

安全设计对应图里从左到右的**三道门**：

| 图中位置 | 是谁 | 手里拿着什么 |
|------|------|-----------|
| **左边（来客）** | 你和你的 AI 客户端 | 你领到的记忆钥匙 `sk-mem-` |
| **中间（关卡）** | MemorySkills 服务端 | 空间钥匙 `sk-service-`，**只留在服务端，绝不发给你** |
| **右边（宝库）** | MemoryBear 记忆库 | 真正存取记忆的 REST 读写接口 |

一句话说清：**你手上的 `sk-mem-` 不会直接连到 MemoryBear**。你的请求先到中间的 MemorySkills，它验过你的记忆钥匙，再用自己那把 `sk-service-` 替你去读写记忆。就像图里那枚红色的空间钥匙——它只待在中间这道门里，从不会交到你手上，这就是这套设计的安全边界。

---

## 三步领到钥匙（你在浏览器里做的部分）

<div align="center">
<img src="assets/onboarding.png" alt="免密登录领取记忆钥匙" width="820" />
</div>

Agent 会帮你打开下面这个登录页，你在浏览器里完成这三步即可：

1. **打开登录页**：在浏览器访问 `<portal>/#/login`（当前即 `https://memoryskills.redbearai.com/#/login`）。
2. **邮箱验证码免密登录**：输入邮箱 → 收验证码 → 提交。新用户自动注册，老用户自动登录，全程无需密码。
3. **复制记忆钥匙**：登录成功后同一页面切换为「凭证卡」，点击 `api_key` 行右侧的 **复制** 按钮，拿到的就是纯净的 `sk-mem-...`。

> 同一账号无论登录多少次，拿到的都是**同一把钥匙**，不会自动滚动生成新 Key，也就不用担心「再登一次会不会让旧配置失效」。凭证卡上还有「查看记忆图谱」按钮，可随时回看自己已写入的记忆。

领到钥匙后，Skill 会引导 AI 把它安全地写进客户端的用户级 MCP 配置：

```jsonc
{
  "memorybear": {
    "type": "http",
    "url": "https://memoryskills.redbearai.com/v1/mcp/memory/",  // 注意结尾的斜杠
    "headers": {
      "Authorization": "Bearer <你的 sk-mem- 钥匙>"
    }
  }
}
```

不同客户端字段略有差异（VS Code 用 `servers`，Cursor 用 `transport` 而非 `type`），Skill 会自动适配。

---

## 两式记忆神通

接入完成后，你的 AI 客户端会多出两个工具：

<div align="center">
<img src="assets/memory-tools.png" alt="读写双诀：read_memory / write_memory" width="820" />
</div>

### 回忆 · `read_memory`

当问题依赖历史偏好、过往决策或长期上下文时，从记忆深处取回相关内容。

```text
read_memory(message="<与当前问题直接相关的自然语言查询>", search_switch="express")
```

- `message`（必填）：查询文本，**不能为空**。
- `search_switch`（可选，默认 `express`）：检索深度档位。
  - `express`：快速召回，适合大多数场景（默认）。
  - `deep`：更完整扫描历史，延迟稍高。
  - `research`：最深档，延迟最高，仅用于研究性回顾。

### 记住 · `write_memory`

当你明确要求记住，或提供了值得跨会话保存的稳定偏好、长期决策时，把它刻入记忆。

```text
write_memory(message="<简洁、完整、可独立理解的记忆>")
```

- `message`（必填）：单条记忆，**1–8000 字符**，超长会被拒绝。
- 每把钥匙每分钟最多 **30 次** 写入，触发限流会返回错误。
- 写入是**异步提交**，成功只表示任务已受理，不代表立即可检索。

---

## 安全边界（务必遵守）

把 API Key 当密码对待。这套 Skill 在设计上遵循以下红线：

- **不回显完整 Key**：不在回复、日志、命令输出或错误信息中打印完整钥匙。
- **不落盘到仓库**：不把 Key 写进项目目录、代码仓库、规则文件、测试文件或历史记录，优先用客户端的 Secret / 环境变量引用。
- **不碰剪贴板**：只有你明确确认「已复制并允许读取」后才读取一次。
- **不入记忆**：验证码、API Key、Token 等敏感凭证一律不写进记忆。
- **HTTPS 优先**：HTTP / 局域网地址仅限受信任的开发环境；投产必须走 HTTPS。
- **不传身份参数**：不接收、生成或传递 `workspace_id` / `end_user_id` / `other_id`，身份已绑定在 Session 或 Key 上。

---

## 完成标准

只有同时满足以下条件，才算接入完成：

- [ ] 已在浏览器 `/#/login` 通过邮箱验证码完成注册或登录
- [ ] 已从凭证卡复制或安全交付 `sk-mem-` 钥匙
- [ ] 钥匙只存在于用户批准的安全存储或用户级配置
- [ ] MCP URL 来自 `/api/runtime-config` 的 `skills_base_url` 拼接 `/v1/mcp/memory/`
- [ ] `memorybear` MCP 已连接成功
- [ ] 工具列表中同时出现 `read_memory` 与 `write_memory`
- [ ] 项目文件与 Git Diff 中不存在任何 Key

---

## 常见故障速查

| 现象 | 原因与处理 |
|------|-----------|
| 发码返回 429 | 邮箱/IP 触发发码限流，等窗口期过后重试，别反复点「发送验证码」 |
| `write_memory` 返回 429 | 写入超频（30 次/分钟），停手稍后再试，**不要**自动重试循环 |
| `ToolError: message 长度超限` | 单条记忆超 8000 字符，先摘要或拆条写入 |
| `ToolError: search_switch 无效` | 只接受 `express` / `deep` / `research` 三值 |
| MCP 返回 401 | Header 缺 `Authorization` 或 Key 失效，回登录页重新复制同一把 Key |
| MCP 返回 403 | 不要加 `other_id` 绕过，应排查 Key 归属或权限 |
| 登录返回 403 `CSRF_ORIGIN_INVALID` | 浏览器 Origin 与服务器白名单协议不一致（如 HTTPS vs HTTP），请管理员核对 |
| 写入后立即读不到 | 异步处理，稍后重试，不要重复批量写入 |

---

<div align="center">



</div>
