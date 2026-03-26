# Question

可以通过setting file来指定吗

---

# Answer

可以的，但方式稍有不同：

| 需要‑控制的东西 | 通过 *Settings*（`settings.json`）可以直接配置吗 | 如果可以，在哪个字段 / 哪段代码里生效 |
|----------------|----------------------------------------------|-----------------------------------|
| **默认模型**（如 ChatGPT‑4、Claude‑2、Gemini 等） | ✅ 直接可通过 Settings 配置 | <ul><li>`AgentSettings.default_model`（在 *Settings* 中叫 **`agent.default_model`** 存放 **`LanguageModelSelection`**。</li><li>在 `crates/agent_ui/src/agent_ui.rs` 的 `init_language_model_settings`（行号 **≈ 501‑560**）里读取 `settings.default_model`，把它转换为 `language_model::SelectedModel`，并调用 `LanguageModelRegistry::global(cx).select_default_model(..)`。</li></ul> |
| **默认 *Agent***（打开 Agent 面板时显示的“Zed Agent / 自定义 Agent”） | ❌ 目前 **没有** 在 `settings.json` 中直接提供的字段 | <ul><li>默认值硬编码在 `crates/agent_ui/src/agent_panel.rs` 的 `selected_agent_type: AgentType::default()`（行 ≈ 1089），该默认是 `AgentType::NativeAgent`（即内置的 Zed Agent）。</li><li>如果用户之前切换过其他 Agent，面板会把它写进 KV‑store（键名 `agent_panel`），在 `serialize` 方法里写入 `selected_agent: Some(selected_agent_type)`（行 ≈ 789‑795），随后在 `load` 时读取并恢复（行 ≈ 877‑883）。</li></ul>

## 1. 如何在 Settings 中指定默认模型

```json
{
  // …其他 Zed 设置…
  "agent": {
    // 只写这一行即可覆盖默认模型
    "default_model": {
      "provider": "anthropic",   // provider id，如 openai、google、ollama …等
      "model": "claude-2.1"       // 具体模型名称
    }

    // 下面的字段分别控制其它场景的模型（可选）
    // "inline_assistant_model": { "provider": "...", "model": "..." },
    // "commit_message_model":    { "provider": "...", "model": "..." },
    // "thread_summary_model":   { "provider": "...", "model": "..." }
  }
}
```

* `settings.json` 位于用户的 **Zed 配置目录**（`~/.config/zed/settings.json` 或在 macOS/Windows 对应路径），编辑后保存，Zed 会在下次启动或在 `init_language_model_settings` 被调用时读取它。

* 当 `AgentSettings::default_model` 为 `None` 时，Zed 会使用语言模型注册表的默认提供者（通常是第一个被加载且可用的 provider），所以显式写上上面的字段可以确保每次打开 Zed 时都使用您想要的模型。

## 2. “默认 Agent” 目前没有 Settings 入口

* **现状**：代码里只在 `AgentPanel::new` 时把 `selected_agent_type` 初始化为 `AgentType::default()`（即 `NativeAgent`），而后面的 “记住上一次使用的 Agent” 通过 KV‑store（`KeyValueStore`）实现。没有任何 `AgentSettings` 或 `Settings` 字段来控制它。

* **如果真的想通过配置文件指定**，有两种可行的办法：

  1. **手动写入 KV‑store**\
     只要在用户主目录的 Zed 数据库（Zed 用 `KeyValueStore` 存在 `~/.config/zed/kvp` 之类的地方）里写入键 `agent_panel`，把 `selected_agent` 字段设为您想要的 `AgentType`（例如 `{"selected_agent":{"custom":{"name":"my‑custom‑agent"}}}`），下次启动时 Zed 会直接读取它并把面板打开到该 Agent。此方式不建议手动编辑，除非您对内部存储非常熟悉。

  2. **在插件里提供一个 “默认Agent” 设置**（最干净的方式）\
     在 Zed 自己的 `AgentSettings` 结构里加入一个字段（例如 `default_agent: Option<Agent>`），并在 `AgentPanel::new`（或 `load`）里先检查这个字段，如果有值就覆盖 `selected_agent_type`。这需要一次小的代码改动并重新编译 Zed，但之后您就可以像下面这样在 `settings.json` 中配置：

```json
{
  "agent": {
    "default_agent": {
      "custom": { "name": "my-custom-agent" }
    },
    "default_model": { "provider": "openai", "model": "gpt-4o" }
  }
}
```

  - **实现要点（示例）**

    ```rust
    // 在 crates/agent_ui/src/agent.rs（或相应的 settings 定义文件）添加字段
    #[derive(Deserialize, Serialize, Debug, Clone)]
    pub struct AgentSettings {
        // …已有字段…
        pub default_agent: Option<Agent>,   // ← 新增
        // …
    }

    // 在 AgentPanel::new 或 load 中
    if let Some(default_agent) = AgentSettings::get_global(cx).default_agent {
        panel.selected_agent_type = default_agent.into(); // 把 custom / native 转成 AgentType
    } else {
        // 继续使用原来的默认 NativeAgent
    }
    ```

这样就可以像配置其他 Zed 选项一样，统一管理默认 Agent 与模型。

---

**小结**

| 目标 | 现有方式 | 通过 Settings 直接配置？ |
|------|-----------|------------------------|
| **默认模型** | `AgentSettings.default_model` → `init_language_model_settings` | ✅（在 `settings.json` 的 `agent.default_model`） |
| **默认 Agent** | 代码硬编码 `NativeAgent`，或从 KV‑store 读取上一次使用的值 | ❌（目前没有 Settings 支持）<br>**解决方案**：自行在 `settings.json` 中写 `agent.default_model`（模型）; 若想控制默认 Agent，建议在代码里添加 `default_agent` 字段或手动编辑 KV‑store。

---

# 源码验证结果（2026-03-26）

## 验证结论

回答**大体正确**，但存在一些细节上的**过时或不准确**之处。以下是逐项验证：

---

## ✅ 正确的部分

### 1. 默认模型 — 可以通过 `settings.json` 配置
- **验证通过**。`AgentSettings` 结构体（`crates/agent_settings/src/agent_settings.rs` L32）确实有 `default_model: Option<LanguageModelSelection>` 字段
- 在 `settings.json` 中通过 `agent.default_model` 设置

### 2. `init_language_model_settings` 读取 settings 并应用
- **验证通过**。`crates/agent_ui/src/agent_ui.rs` L501‑557 中的 `init_language_model_settings` 和 `update_active_language_model_from_settings` 确实读取 `AgentSettings` 的 `default_model`，并调用 `registry.select_default_model()`

### 3. 默认 Agent — 没有 `settings.json` 入口
- **验证通过**。搜索整个 crates 目录，不存在 `default_agent` 字段
- `AgentPanel::new`（`crates/agent_ui/src/agent_panel.rs` L1089）确实初始化为 `AgentType::default()`，即 `NativeAgent`

### 4. Agent 选择通过 KV‑store 持久化
- **验证通过**。`serialize` 方法（`agent_panel.rs` L786‑795）将 `selected_agent_type` 存入 `KeyValueStore`
- `load` 方法（`agent_panel.rs` L877‑881）从中读取并恢复

---

## ⚠️ 需要注意 / 修正的部分

### 1. `LanguageModelSelection` 结构变化
回答中写的 JSON 格式为：
```json
{
  "provider": "anthropic",
  "model": "claude-2.1"
}
```
但实际代码中 `LanguageModelSelection`（`crates/settings_content/src/agent.rs` L312‑319）还支持两个额外字段：
- `enable_thinking: bool`（默认 false）
- `effort: Option<String>`

这不影响基本使用（它们有默认值），但如果想启用 thinking 模式，需要额外配置。

### 2. 代码路径说法
- 回答提到 `crates/agent_ui/src/agent.rs` —— 实际上设置定义在 `crates/agent_settings/src/agent_settings.rs` 和 `crates/settings_content/src/agent.rs`
- 回答提到 `AgentSettings.default_model` —— 实际上 `AgentSettings` 在 `crates/agent_settings/` 而非 `crates/agent_ui/`

### 3. 行号偏差
回答中提到的行号（如 501‑560，1089，789‑795，877‑883）与实际代码**基本吻合**（实际行号略有偏移但对应同样的逻辑）。

---

## 🔧 如何设置默认模型和 Agent

### 设置默认模型

在 `settings.json` 中添加以下配置即可：

```json
{
  "agent": {
    "default_model": {
      "provider": "anthropic",
      "model": "claude-sonnet-4-20250514"
    }
  }
}
```

可用的 `provider` 标识符包括：
`anthropic`、`openai`、`google`、`ollama`、`deepseek`、`openrouter`、`copilot_chat`、`lmstudio`、`mistral`、`x_ai`、`zed.dev`、`amazon-bedrock`、`vercel`、`vercel_ai_gateway` 等。

还可以为不同场景单独指定模型：

```json
{
  "agent": {
    "default_model": {
      "provider": "anthropic",
      "model": "claude-sonnet-4-20250514"
    },
    "inline_assistant_model": {
      "provider": "anthropic",
      "model": "claude-sonnet-4-20250514"
    },
    "commit_message_model": {
      "provider": "google",
      "model": "gemini-2.0-flash"
    },
    "thread_summary_model": {
      "provider": "google",
      "model": "gemini-2.0-flash"
    }
  }
}
```

### 设置默认 Agent

目前 **无法** 通过 `settings.json` 指定默认 Agent。代码中不存在 `default_agent` 字段。

**现有机制**：
1. 首次启动默认使用 `NativeAgent`（内置 Zed Agent）
2. 当你在 UI 中切换到其他 Agent（如自定义 Agent Server），Zed 会自动将选择持久化到 KV‑store
3. 下次启动时，Zed 会自动恢复上次使用的 Agent

**实际操作**：只需在 Zed 的 Agent 面板中手动切换一次到你想要的 Agent，之后每次启动都会自动使用该 Agent。

如果需要通过配置文件控制，需要修改 Zed 源码（如回答中建议的添加 `default_agent` 字段），目前上游没有这个功能。