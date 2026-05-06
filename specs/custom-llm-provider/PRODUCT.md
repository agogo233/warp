# 自定义 LLM Provider 功能 - 产品规格文档

## 概述
允许 Warp 终端用户配置自定义的 LLM Provider（如 OpenRouter、本地 LLM 服务等），支持自定义 API URL 和 API Key，突破当前仅支持预设提供商（OpenAI、Anthropic、Google、Xai）的限制。

## 问题陈述

### 当前痛点
1. **提供商锁定**：Warp 目前仅支持 4 个预设 LLM 提供商，用户无法使用其他兼容 OpenAI API 的服务
2. **无法使用本地模型**：用户无法配置本地 LLM 服务（如 Ollama、LM Studio）
3. **无法使用 OpenRouter 等聚合服务**：许多用户使用 OpenRouter 来访问多个模型
4. **企业合规限制**：部分企业要求使用自托管的 LLM 服务，当前无法实现

### 现状分析
通过代码调查发现：
- `LLMProvider` 枚举（`app/src/ai/llms.rs:104-110`）仅定义 5 种提供商：OpenAI、Anthropic、Google、Xai、Unknown
- 模型配置通过 GraphQL API 从服务器下发（`get_feature_model_choices`），非纯本地配置
- BYOK（Bring Your Own Key）功能存在，但仅适用于内置提供商的 API Key 替换
- CLI Agents（Codex、Claude Code）支持部分自定义 endpoint（如 `openai_base_url`），但未开放给普通用户配置
- 核心 AI 功能（Agent Mode、Blocklist AI）通过 Warp 服务器代理，自定义 endpoint 需要服务器端支持

## 目标

### 主要目标
- 允许用户添加自定义 LLM Provider（OpenAI API 兼容）
- 支持配置自定义 Base URL 和 API Key
- 在 CLI Agent 中优先支持自定义 Provider
- 提供用户友好的配置界面

### 次要目标
- 支持自定义 Provider 的模型列表配置（自动获取或手动配置）
- 在核心 AI 功能中探索自定义 Provider 支持的可能性
- 支持多个自定义 Provider 配置

## 非目标
- 不支持非 OpenAI 兼容的 API（如需要特殊认证的提供商）
- 不修改服务器端逻辑（第一阶段）
- 不提供模型性能测试或比较功能
- 不自动检测 Provider 类型（用户需手动配置）

## 用户场景

### 场景 1：使用 OpenRouter
**用户**：开发者，希望使用 OpenRouter 访问多个模型
**流程**：
1. 在设置中添加自定义 Provider
2. 名称：`OpenRouter`
3. Base URL：`https://openrouter.ai/api/v1`
4. API Key：粘贴 OpenRouter API Key
5. 模型列表：留空（自动从 `/models` endpoint 获取）
6. 在 CLI Agent 中选择使用 OpenRouter

### 场景 2：使用本地 LLM（Ollama）
**用户**：隐私意识强的开发者，使用本地 Ollama 运行 Llama 3
**流程**：
1. 在设置中添加自定义 Provider
2. 名称：`Ollama Local`
3. Base URL：`http://localhost:11434/v1`
4. API Key：留空（本地服务无需 Key）
5. 手动配置模型：`llama3`、`mistral` 等
6. 完全离线使用 Warp AI 功能

### 场景 3：企业自托管 LLM
**用户**：企业用户，公司使用自托管的 LLM 服务
**流程**：
1. IT 部门提供 Base URL 和 API Key
2. 用户配置自定义 Provider
3. 所有请求发送到企业内部服务，数据不出内网

## 成功标准
1. 用户能成功添加、编辑、删除自定义 Provider 配置
2. 自定义 Provider 能在 CLI Agent（Codex）中正常使用
3. 配置界面直观易用，有清晰的验证反馈
4. API Key 安全存储（通过现有的 `ManagedSecretValue` 机制）
5. 不影响现有预设提供商的功能
6. 配置数据不云同步（避免敏感信息泄露）

## 验证计划
- **单元测试**：配置序列化和反序列化、UI 逻辑
- **集成测试**：CLI Agent 使用自定义 Provider 发送请求
- **手动测试**：
  - 使用 OpenRouter 发送真实请求
  - 使用 Ollama 本地服务发送请求
  - 验证错误场景（无效 URL、无效 Key）
- **安全测试**：验证 API Key 是否正确存储和读取

## 开放问题
1. **核心 AI 功能的支持**：是否应该/可以在第一阶段支持核心 AI 功能的自定义 Provider？（需要服务器端修改）
2. **模型列表获取**：是否应该自动调用 `/models` endpoint 获取模型列表？如何处理网络错误？
3. **配置同步**：是否应该允许用户选择将自定义 Provider 配置云同步？（安全考虑）
4. **UI 复杂度**：配置界面应该简单（仅 URL + Key）还是支持高级选项（超时、自定义 header 等）？

## 参考资料
- 现有 BYOK 实现：`app/src/settings/ai.rs`
- CLI Agent 自定义 endpoint 示例：`app/src/ai/agent_sdk/driver/harness/codex.rs:433-437`
- 设置 UI 实现：`app/src/settings_view/ai_page.rs`
- GraphQL 模型查询：`crates/graphql/src/api/workspace.rs:65-77`
