# 自定义 LLM Provider 功能 - 技术规格文档

## 深度架构分析

### 当前架构概述

Warp 的 AI 功能分为两大路径：

```
┌─────────────────────────────────────────────────────────────┐
│                        Warp 终端客户端                        │
├─────────────────────────────────────────────────────────────┤
│  核心 AI 功能                    │   CLI Agent (独立进程)      │
│  - Agent Mode                    │   - Codex                 │
│  - Blocklist AI                  │   - Claude Code           │
│  - Predict/Autosuggestions      │   - Gemini                │
├─────────────────────────────────────────────────────────────┤
│          通过 Warp 服务器代理                        │
│   请求 → warp-server → LLM Provider API                    │
│   (模型选择由服务器下发的 `LLMInfo` 控制)                  │
└─────────────────────────────────────────────────────────────┘
         │                              │
         │                              │ 直接调用外部 API
         ▼                              ▼
   服务器端控制                    CLI Agent 进程
   (需修改服务器)                  (可自定义 endpoint)
```

### 关键数据流分析

#### 1. 模型选择流程
```
用户选择模型 → LLMId → 查找 LLMInfo → 构建请求 → 发送到服务器
```

**关键代码点**：
- `LLMProvider` 枚举定义：`app/src/ai/llms.rs:104-110`
- 模型数据来源：GraphQL `get_feature_model_choices` 查询
- 模型缓存：`ModelsByFeature` 结构（`app/src/ai/llms.rs:410`）

#### 2. CLI Agent 请求流程
```
AgentDriver → Harness (codex.rs/claude_code.rs) → 环境变量/配置文件 → 外部 API
```

**关键发现**：
- Codex harness 支持 `openai_base_url` 配置（`codex.rs:433`）
- 通过 `~/.codex/config.toml` 配置文件控制
- API Key 通过 `ManagedSecretValue` 机制管理

#### 3. BYOK (Bring Your Own Key) 机制
```
用户设置 → AISettings → ApiKeyManager → ManagedSecretValue → 请求时使用
```

**配置路径**：`cloud_platform.third_party_api_keys.*`（`app/src/settings/ai.rs:1017-1052`）

### 当前限制的根本原因

1. **服务器端控制**：核心 AI 功能的模型列表和路由由服务器控制，客户端无法添加未列出的 Provider
2. **枚举限制**：`LLMProvider` 是封闭枚举，无法动态扩展
3. **配置缺失**：没有 `base_url` 的用户配置入口
4. **UI 限制**：设置页面仅支持预设提供商的 API Key 输入

## 技术方案设计

### 方案对比

| 方案 | 优点 | 缺点 | 复杂度 | 推荐 |
|------|------|------|--------|------|
| **A. 仅扩展 CLI Agent** | 实现简单，不影响核心流程 | 核心 AI 功能无法使用 | 低 | ✅ 第一阶段 |
| **B. 完全扩展（含核心 AI）** | 功能完整 | 需要服务器端修改，复杂度高 | 高 | 第二阶段 |
| **C. 本地代理模式** | 不依赖服务器，完全本地控制 | 需要维护代理服务，用户体验差 | 中 | ❌ 不推荐 |
| **D. 插件系统** | 扩展性强，未来可复用 | 过度设计，当前需求简单 | 极高 | ❌ 不推荐 |

### 推荐方案：分阶段实施

#### 阶段 1：CLI Agent 支持（推荐优先）
**目标**：让 Codex、Claude Code 等 CLI Agent 支持自定义 Provider

**优势**：
- 不需要服务器端修改
- 利用现有的 `openai_base_url` 机制
- 风险低，影响范围小
- 快速交付价值

**实施重点**：
1. 扩展 `LLMProvider` 枚举（添加 `CustomOpenAI` 变体）
2. 创建设置 UI 让用户输入自定义 Provider 配置
3. 修改 CLI Agent driver 使用自定义配置
4. 通过 `ManagedSecretValue` 安全存储 API Key

#### 阶段 2：核心 AI 功能探索（未来）
**先决条件**：服务器端支持或本地请求路由

**可能方向**：
- 与 Warp 团队讨论服务器端添加自定义 Provider 支持
- 或实现客户端直接调用（绕过服务器代理）
- 需要深入评估安全性和计费影响

## 详细技术设计

### 1. 数据模型扩展

#### 1.1 扩展 `LLMProvider` 枚举

**文件**：`app/src/ai/llms.rs:104-110`

当前状态：
```rust
pub enum LLMProvider {
    OpenAI,
    Anthropic,
    Google,
    Xai,
    Unknown,
}
```

扩展后：
```rust
pub enum LLMProvider {
    OpenAI,
    Anthropic,
    Google,
    Xai,
    /// 自定义 OpenAI 兼容的 Provider
    CustomOpenAI {
        id: String,           // 唯一标识，如 "custom-openrouter"
        display_name: String,  // 显示名称，如 "OpenRouter"
        base_url: String,       // 自定义 endpoint，如 "https://openrouter.ai/api/v1"
    },
    Unknown,
}
```

**序列化考虑**：
- 使用 `#[serde(tag = "type", content = "data")]` 或类似方式处理枚举变体的序列化
- 需要修改 `LLMProvider` 的 `Serialize`/`Deserialize` 实现

#### 1.2 自定义 Provider 配置结构

**新文件**：`app/src/ai/custom_provider.rs`

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CustomProviderConfig {
    pub id: String,                      // 唯一标识
    pub display_name: String,            // 用户显示名称
    pub base_url: String,                // API base URL
    pub api_key_secret_key: String,      // 对应 ManagedSecretValue 的 key
    pub models: Vec<CustomModelConfig>,  // 模型列表（可选）
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub updated_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CustomModelConfig {
    pub id: String,           // 模型 ID，如 "anthropic/claude-3.5-sonnet"
    pub display_name: String, // 显示名称
}

impl CustomProviderConfig {
    /// 生成用于 ManagedSecretValue 的 key
    pub fn secret_key(&self) -> String {
        format!("custom_provider_{}_api_key", self.id)
    }
}
```

#### 1.3 扩展 `ManagedSecretValue`

**文件**：`warp_managed_secrets` crate（需要确认准确位置）

当前支持的变体：
```rust
pub enum ManagedSecretValue {
    RawValue { value: String },
    AnthropicApiKey { api_key: String },
    AnthropicBedrockApiKey { ... },
    AnthropicBedrockAccessKey { ... },
}
```

**选项 1**：复用 `RawValue`（推荐，无需修改 crate）
- 自定义 Provider 的 API Key 作为 `RawValue` 存储
- 通过命名约定区分：`custom_provider_{id}_api_key`

**选项 2**：添加新变体
```rust
pub enum ManagedSecretValue {
    // ... 现有变体
    CustomProviderApiKey { provider_id: String, api_key: String },
}
```

### 2. 设置系统集成

#### 2.1 添加设置项

**文件**：`app/src/settings/ai.rs`

在 `define_settings_group!` 宏中添加：

```rust
custom_llm_providers: CustomLLMProviders {
    type: Vec<CustomProviderConfig>,
    default: vec![],
    supported_platforms: SupportedPlatforms::ALL,
    sync_to_cloud: SyncToCloud::Never,  // 不云同步（包含敏感信息）
    private: true,
    toml_path: "cloud_platform.third_party_api_keys.custom_providers",
    description: "Custom LLM provider configurations.",
}
```

**注意**：需要为 `CustomProviderConfig` 实现 `settings_value::SettingsValue` trait。

#### 2.2 设置读取和验证

添加辅助函数：
```rust
// app/src/settings/ai.rs

impl AISettings {
    /// 获取所有自定义 Provider 配置
    pub fn custom_llm_providers(&self) -> Vec<CustomProviderConfig> {
        self.custom_llm_providers.clone()
    }
    
    /// 根据 ID 查找自定义 Provider
    pub fn find_custom_provider(&self, id: &str) -> Option<CustomProviderConfig> {
        self.custom_llm_providers.iter().find(|p| p.id == id).cloned()
    }
    
    /// 验证 Provider 配置
    pub fn validate_custom_provider(provider: &CustomProviderConfig) -> Result<(), String> {
        // 验证 base_url 格式
        if !provider.base_url.starts_with("http://") && !provider.base_url.starts_with("https://") {
            return Err("Base URL must start with http:// or https://".to_string());
        }
        
        // 验证 ID 格式（不允许空格、特殊字符等）
        if provider.id.contains(|c: char| !c.is_alphanumeric() && c != '-' && c != '_') {
            return Err("Provider ID can only contain alphanumeric characters, hyphens, and underscores".to_string());
        }
        
        Ok(())
    }
}
```

### 3. UI 实现

#### 3.1 设置页面结构

**文件**：`app/src/settings_view/ai_page.rs`

在 `AISubpage` 枚举中添加新页面：
```rust
pub enum AISubpage {
    WarpAgent,
    Profiles,
    Knowledge,
    ThirdPartyCLIAgents,
    /// 自定义 LLM Provider 配置页面
    CustomProviders,  // 新增
}
```

#### 3.2 UI 组件设计

**自定义 Provider 列表视图**：
```
┌─────────────────────────────────────────────────┐
│  Custom LLM Providers                           │
├─────────────────────────────────────────────────┤
│  [+ Add Provider]                               │
│                                                 │
│  ┌─────────────────────────────────────────────┐ │
│  │ OpenRouter                      [Edit] [Delete]│
│  │ https://openrouter.ai/api/v1                │ │
│  │ Models: 15 configured                       │ │
│  └─────────────────────────────────────────────┘ │
│                                                 │
│  ┌─────────────────────────────────────────────┐ │
│  │ Ollama Local                   [Edit] [Delete]│
│  │ http://localhost:11434/v1                   │ │
│  │ Models: llama3, mistral                     │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

**添加/编辑 Provider 表单**：
```
┌─────────────────────────────────────────────────┐
│  Add Custom LLM Provider                       │
├─────────────────────────────────────────────────┤
│  Name: [__________________________________]   │
│  (显示名称，如 "OpenRouter")                      │
│                                                 │
│  Provider ID: [______________________________]   │
│  (唯一标识，仅允许字母、数字、横线、下划线)        │
│                                                 │
│  Base URL: [________________________________]  │
│  (API endpoint，如 https://openrouter.ai/api/v1) │
│                                                 │
│  API Key: [________________________________]  │
│  (留空表示无需认证，如本地服务)                    │
│                                                 │
│  Models (可选):                                  │
│  [ ] Auto-fetch from /models endpoint           │
│  [+] Add model manually                         │
│      Model ID: [____________] Display: [_____]  │
│                                                 │
│  [Test Connection] [Cancel] [Save]              │
└─────────────────────────────────────────────────┘
```

#### 3.3 实现要点

使用 warpui 框架构建 UI：
- 使用 `Flex`、`Container`、`Text`、`Button` 等基础组件
- 表单使用 `EditorView` 或自定义输入组件
- 列表使用 `Flex` + `Container` 构建
- 添加表单验证逻辑，实时显示错误提示

### 4. CLI Agent 集成

#### 4.1 修改 Codex Driver

**文件**：`app/src/ai/agent_sdk/driver/harness/codex.rs`

扩展 `prepare_codex_environment_config` 函数：

```rust
fn prepare_codex_environment_config(
    working_dir: &Path,
    system_prompt: Option<&str>,
    secrets: &HashMap<String, ManagedSecretValue>,
    custom_providers: &[CustomProviderConfig],  // 新增参数
) -> Result<()> {
    let home_dir = dirs::home_dir().ok_or_else(|| anyhow::anyhow!("could not determine home directory"))?;
    let codex_dir = home_dir.join(CODEX_CONFIG_DIR);
    
    // ... 现有逻辑（系统提示、API Key）...
    
    // 为自定义 Provider 设置环境变量或配置文件
    if let Some(custom_provider) = custom_providers.first() {  // 简化：先用第一个
        // 设置 OPENAI_API_KEY
        if let Some(ManagedSecretValue::RawValue { value }) = secrets.get(&custom_provider.secret_key()) {
            prepare_codex_auth(&codex_dir.join(CODEX_AUTH_FILE_NAME), value)?;
        }
        
        // 设置 openai_base_url
        prepare_codex_config_toml(&codex_dir.join(CODEX_CONFIG_TOML_FILE_NAME), working_dir, &custom_provider.base_url)?;
    }
    
    Ok(())
}
```

**注意**：需要修改调用处，传入自定义 Provider 配置。

#### 4.2 修改模型选择逻辑

**文件**：`app/src/ai/llms.rs`

在 `ModelsByFeature` 中添加自定义 Provider 模型：

```rust
impl ModelsByFeature {
    /// 合并自定义 Provider 的模型
    pub fn with_custom_providers(mut self, providers: &[CustomProviderConfig]) -> Self {
        for provider in providers {
            let custom_llm_info = LLMInfo {
                display_name: provider.display_name.clone(),
                base_model_name: provider.display_name.clone(),
                id: LLMId::from(format!("custom-{}", provider.id)),
                reasoning_level: None,
                usage_metadata: LLMUsageMetadata {
                    request_multiplier: 1,
                    credit_multiplier: None,
                },
                description: Some(format!("Custom provider: {}", provider.base_url)),
                disable_reason: None,
                vision_supported: false,  // 默认不支持，用户可手动配置
                spec: None,
                provider: LLMProvider::CustomOpenAI {
                    id: provider.id.clone(),
                    display_name: provider.display_name.clone(),
                    base_url: provider.base_url.clone(),
                },
                host_configs: HashMap::new(),
                discount_percentage: None,
                context_window: LLMContextWindow::default(),
            };
            
            // 添加到 agent_mode 和 cli_agent
            self.agent_mode.choices.push(custom_llm_info.clone());
            if let Some(ref mut cli_agent) = self.cli_agent {
                cli_agent.choices.push(custom_llm_info);
            }
        }
        self
    }
}
```

### 5. 测试策略

#### 5.1 单元测试

**测试文件**：
- `app/src/ai/custom_provider_tests.rs`
- 扩展 `app/src/settings/ai_tests.rs`

**测试用例**：
```rust
#[test]
fn test_custom_provider_config_serialization() {
    let config = CustomProviderConfig { ... };
    let json = serde_json::to_string(&config).unwrap();
    let deserialized: CustomProviderConfig = serde_json::from_str(&json).unwrap();
    assert_eq!(config.id, deserialized.id);
}

#[test]
fn test_custom_provider_validation() {
    let mut provider = CustomProviderConfig { ... };
    provider.base_url = "invalid-url".to_string();
    assert!(AISettings::validate_custom_provider(&provider).is_err());
    
    provider.base_url = "https://openrouter.ai/api/v1".to_string();
    assert!(AISettings::validate_custom_provider(&provider).is_ok());
}
```

#### 5.2 集成测试

**测试文件**：`crates/integration/src/test/custom_llm_provider.rs`

**测试场景**：
1. 添加自定义 Provider 配置
2. 启动 Codex Agent 并使用自定义 Provider
3. 验证请求发送到正确的 endpoint
4. 测试错误场景（无效 URL、无效 Key）

#### 5.3 手动测试清单

- [ ] 在设置中添加自定义 Provider（OpenRouter）
- [ ] 配置 API Key 和 Base URL
- [ ] 在 CLI Agent 中选择自定义 Provider
- [ ] 发送测试请求，验证返回结果
- [ ] 测试本地服务（Ollama，无 API Key）
- [ ] 测试编辑和删除 Provider
- [ ] 测试表单验证（无效 URL、重复 ID）
- [ ] 测试 API Key 安全存储（不在日志中泄露）

## 风险评估与缓解措施

### 风险 1：服务器端限制
**风险**：核心 AI 功能无法使用自定义 Provider，因为请求通过服务器代理
**影响**：高（功能不完整）
**缓解**：
- 第一阶段仅支持 CLI Agent，明确告知用户
- 第二阶段与服务器团队合作，探索支持方案
- 文档中明确说明哪些功能支持自定义 Provider

### 风险 2：API 兼容性问题
**风险**：用户配置的自定义 Provider 不完全兼容 OpenAI API
**影响**：中（部分功能异常）
**缓解**：
- 提供 "Test Connection" 按钮，验证基本兼容性
- 文档中说明兼容要求
- 错误提示中给出诊断建议

### 风险 3：安全风险
**风险**：
- API Key 泄露（日志、错误报告）
- 恶意 URL（SSRF 攻击）
**影响**：高（安全事故）
**缓解**：
- 复用现有的 `ManagedSecretValue` 安全存储机制
- 配置不云同步（`SyncToCloud::Never`）
- URL 验证：仅允许 http/https，禁止内网地址（可选）
- 文档中提醒用户注意 API Key 安全

### 风险 4：用户体验问题
**风险**：配置复杂，用户不理解如何填写
**影响**：中（用户流失）
**缓解**：
- 提供预设模板（OpenRouter、Ollama、LM Studio 等）
- 表单中添加详细的帮助文本
- 提供 "Test Connection" 功能
- UI 中显示配置示例

### 风险 5：维护负担
**风险**：未来 `LLMProvider` 枚举或 API 变更导致兼容性问题
**影响**：中（技术债务）
**缓解**：
- 代码模块化，将自定义 Provider 逻辑隔离
- 添加充分的单元测试和集成测试
- 文档中记录设计决策和依赖点

## 后续优化方向

### 短期（1-3 个月）
1. **预设模板**：为常见 Provider（OpenRouter、Ollama、LM Studio、LocalAI）提供一键配置
2. **模型自动发现**：调用 `/models` endpoint 自动获取可用模型列表
3. **连接测试**：实现 "Test Connection" 功能，验证配置正确性
4. **错误诊断**：当请求失败时，提供更详细的错误信息和解决建议

### 中期（3-6 个月）
1. **多个自定义 Provider 支持**：CLI Agent 支持在多个自定义 Provider 间切换
2. **核心 AI 功能探索**：与团队讨论如何支持核心 AI 功能的自定义 Provider
3. **使用统计**：显示自定义 Provider 的请求次数、成功率等
4. **导出/导入配置**：方便用户分享或备份配置

### 长期（6+ 个月）
1. **插件生态系统**：如果需求增长，考虑更通用的插件系统
2. **本地 LLM 优化**：针对本地服务（Ollama）优化体验（如自动启动、状态检测）
3. **企业功能**：支持企业级需求（SSO、审计日志、合规检查）

## 端到端流程示例

### 用户配置 OpenRouter 并使用

```
1. 用户打开设置 → AI → Custom Providers
2. 点击 "Add Provider"
3. 填写表单：
   - Name: OpenRouter
   - Provider ID: openrouter
   - Base URL: https://openrouter.ai/api/v1
   - API Key: sk-or-v1-...（用户粘贴）
   - Models: 选择 "Auto-fetch"
4. 点击 "Test Connection" → 显示 "Connected successfully"
5. 点击 "Save"
6. 打开终端，启动 Codex agent
7. 在 agent 设置中选择 "OpenRouter" 作为 Provider
8. 发送请求 → Codex 使用 OpenRouter endpoint 和 API Key
9. 收到响应，功能正常
```

### 数据流

```
用户配置
  ↓
CustomProviderConfig 保存到 settings
  ↓
API Key 通过 ManagedSecretValue 存储
  ↓
启动 CLI Agent (Codex)
  ↓
AgentDriver 读取 custom_providers 配置
  ↓
prepare_codex_environment_config 设置环境变量和配置文件
  ↓
Codex 进程启动，读取 ~/.codex/config.toml
  ↓
openai_base_url = "https://openrouter.ai/api/v1"
  ↓
OPENAI_API_KEY = "sk-or-v1-..."
  ↓
Codex 发送请求到 OpenRouter
  ↓
收到响应，返回给用户
```

## 关键代码变更总结

| 文件 | 变更类型 | 描述 |
|------|----------|------|
| `app/src/ai/llms.rs` | 修改 | 扩展 `LLMProvider` 枚举，添加 `CustomOpenAI` 变体 |
| `app/src/ai/custom_provider.rs` | 新增 | 自定义 Provider 配置结构体和逻辑 |
| `app/src/settings/ai.rs` | 修改 | 添加 `custom_llm_providers` 设置项 |
| `app/src/settings_view/ai_page.rs` | 修改 | 添加 Custom Providers 页面 UI |
| `app/src/ai/agent_sdk/driver/harness/codex.rs` | 修改 | 支持使用自定义 Provider 配置 |
| `app/src/ai/agent_sdk/driver.rs` | 修改 | 传递自定义 Provider 配置给 driver |
| `warp_managed_secrets` | 可能修改 | 如果需要新变体支持自定义 Provider（或使用 RawValue） |

## 验证计划

### 编译和代码质量
```bash
# 编译检查
cargo build --package warp-ai

# Lint 检查
cargo clippy --package warp-ai

# 代码格式化
cargo fmt --package warp-ai -- --check
```

### 自动化测试
```bash
# 运行单元测试
cargo test --package warp-ai custom_provider

# 运行集成测试（如果有）
cargo test --test custom_llm_provider
```

### 手动测试场景
1. **基本功能**：添加、编辑、删除自定义 Provider
2. **兼容性**：使用 OpenRouter、Ollama 测试
3. **错误处理**：无效 URL、无效 Key、网络错误
4. **安全**：API Key 不泄露到日志
5. **UI/UX**：表单验证、错误提示、帮助文本

## 结论

本技术规格文档提供了实现自定义 LLM Provider 功能的完整技术方案。推荐采用**分阶段实施**策略：

**第一阶段**（2-3 周）：
- 实现 CLI Agent 的自定义 Provider 支持
- 完成设置 UI 和配置管理
- 测试和文档

**第二阶段**（未来，取决于服务器团队）：
- 探索核心 AI 功能的自定义 Provider 支持
- 可能需要服务器端修改或新的架构设计

通过本方案，可以快速交付价值（CLI Agent 支持），同时为未来的完整支持奠定基础。
