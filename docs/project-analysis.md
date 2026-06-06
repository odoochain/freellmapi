# FreeLLMAPI 项目核心价值深度分析

## 一、核心问题与解决方案

**痛点**：AI 大模型厂商普遍提供免费额度（Google、Groq、Cerebras 等），但每个厂商单独使用的免费额度很小（几百万 token/月），且各自有不同的 SDK、不同的限流策略、不同的报错格式。开发者想充分利用这些免费额度，需要手动对接十几个 API，维护成本极高。

**方案**：FreeLLMAPI 将所有免费供应商聚合到一个 OpenAI 兼容端点背后，实现"一处配置，全局使用"。

---

## 二、技术架构先进性

### 2.1 多臂老虎机（Multi-Armed Bandit）智能路由

这是项目最核心的技术创新。传统聚合器通常使用固定优先级链（"先试 A，A 失败再试 B"），而 FreeLLMAPI 实现了基于 **Thompson 采样**的概率路由：

```
路由评分 = 凸组合(可靠性 × 速度 × 智能度) × 余量护盾 × 限流护盾
```

- **可靠性**：基于 Beta 分布的 Thompson 采样，自动探索不确定的模型，同时偏向已知可靠的模型
- **速度**：融合吞吐量（指数饱和曲线）和 TTFB（首字节延迟），归一化到 [0,1]
- **智能度**：基于模型能力层级（Frontier > Large > Medium > Small）归一化到 [0,1]
- **余量护盾**：当月免费额度剩余不足 20% 时，分数线性下降，保护模型不被耗尽
- **限流护盾**：实时 429 惩罚机制，每 2 分钟自动衰减，防止向已限流的模型持续发送请求

支持 5 种路由策略：`balanced`（默认）、`smartest`（偏好智能模型）、`fastest`（偏好速度）、`reliable`（偏好稳定性）、`priority`（传统优先级链）、`custom`（用户自定义权重）。

### 2.2 多格式 Provider 适配层

采用统一的适配器模式，将不同供应商的 API 归一化：

| 供应商类型 | 格式 | 适配器 |
|-----------|------|--------|
| OpenAI 兼容 (14个) | OpenAI 标准 | `OpenAICompatProvider` |
| Google Gemini | Gemini 原生格式 | `GoogleProvider`（双向翻译 tools/vision） |
| Cohere | Cohere 兼容格式 | `CohereProvider` |
| Cloudflare | Cloudflare 格式 | `CloudflareProvider` |

对外统一暴露 `/v1/chat/completions`，任何 OpenAI 客户端（LangChain、LlamaIndex、Continue、Cherry Studio 等）零改动接入。

### 2.3 AES-256-GCM 加密密钥存储

所有供应商 API Key 在写入 SQLite 之前使用 AES-256-GCM 加密，密钥仅在内存中解密用于请求。加密密钥通过环境变量注入（生产环境强制要求），开发环境自动生成并持久化到本地数据库。这保证了即使数据库文件泄露，API Key 也不会暴露。

### 2.4 持续式（Constant-Time）安全比较

API Key 认证使用 `crypto.timingSafeEqual` 进行恒定时间比较，防止时序攻击（timing attack）逐字节推测密钥。

### 2.5 指数衰减加权统计

不同于传统的固定 7 天滑动窗口（一个今天开始变差的模型仍保留一周的旧数据），FreeLLMAPI 使用 **指数衰减加权**（半衰期 2 天）来聚合模型性能数据，使近期表现比历史数据权重更高，路由决策更实时。

---

## 三、功能实现创新性

### 3.1 粘性会话（Sticky Sessions）

多轮对话中，自动将后续请求路由到同一模型（30 分钟 TTL），避免中途切换模型导致的"幻觉激增"——这是其他聚合器忽略的关键问题。

### 3.2 智能故障转移

单次请求最多尝试 20 次，遇到 429/5xx/超时自动跳过、冷却、重试。429 惩罚有独立衰减机制，防止模型被永久冻结。

### 3.3 四维限流追踪

每个 `(平台, 模型, Key)` 组合独立追踪：
- RPM（每分钟请求数）
- RPD（每日请求数）
- TPM（每分钟 token 数）
- TPD（每日 token 数）

SQLite 持久化 + 内存缓存双轨制，重启后数据不丢失。

### 3.4 Vision / Tools 感知路由

- 请求包含图片时，**自动排除非视觉模型**，只路由到支持视觉的模型
- 请求包含 tools 时，**自动排除不支持工具调用的模型**
- 无匹配模型时返回明确的 422 错误码，而非静默丢弃

### 3.5 工具参数修复（Tool Args Repair）

对 `/v1/responses` 端点，自动修复供应商返回的格式不规范的 JSON 工具参数，提高跨供应商兼容性。

### 3.6 健康检查自动禁用

每 5 分钟探测所有 Key 的有效性，连续 3 次失败自动禁用。区分传输错误（网络问题）和认证错误（Key 无效），只对后者执行自动禁用。

---

## 四、用户体验优化

### 4.1 统一 API Key

客户端只需一个 `freellmapi-xxx` Token，无需暴露上游供应商 Key。批量轮换供应商 Key 时，客户端零感知。

### 4.2 一键"auto"模型

`model: "auto"` 让系统自动选择最佳模型，支持 Cherry Studio 等不信任空模型字段的客户端。

### 4.3 `X-Routed-Via` 响应头

每次请求返回实际使用的供应商和模型（如 `X-Routed-Via: google/gemini-2.5-flash`），方便调试和审计。

### 4.4 免信用卡即可使用

Kilo、Pollinations、LLM7 三个供应商支持匿名免费访问，无需任何 API Key 即可开始使用。

### 4.5 管理面板

React + Vite 构建的现代化管理界面，支持：
- Key 管理（添加/删除/状态查看）
- Fallback Chain 拖拽排序
- 路由策略切换（5 种策略 + 自定义权重）
- 实时 Analytics（24h/7d/30d 维度，含请求量、成功率、token 数、延迟、预估节省金额）
- Playground 在线测试
- Embeddings 管理

---

## 五、性能表现

- **内存占用**：约 40MB RSS（后端 + 数据库）
- **运行环境**：Node.js 20+，支持 Windows/macOS/Linux/Raspberry Pi
- **流式传输**：所有供应商均支持 SSE 流式输出
- **请求体**：10MB 上限（适配 Code Agent 的大系统提示 + 工具模式）
- **并发**：Express 异步处理，多供应商请求可并行
- **Docker**：多架构镜像（amd64 + arm64），一键部署

---

## 六、市场定位与差异化竞争

### 6.1 与同类产品对比

| 维度 | FreeLLMAPI | LiteLLM | One API | OpenRouter |
|------|-----------|---------|---------|------------|
| 免费额度聚合 | **核心功能** | 需付费后台 | 需付费后台 | 平台本身收费 |
| 智能路由 | **Thompson 采样** | 简单优先级 | 简单优先级 | 无 |
| 故障转移 | **20次自动重试** | 有 | 有 | 无 |
| 粘性会话 | **有(30min)** | 无 | 无 | 无 |
| Vision/Tools路由 | **自动感知** | 无 | 无 | 无 |
| 自托管 | **是** | 是 | 是 | 否 |
| 免信用卡 | **是(3个匿名源)** | N/A | N/A | 否 |
| 加密存储 | **AES-256-GCM** | 无 | 无 | N/A |

### 6.2 独特价值主张

1. **"零成本 AI 基础设施"**：将 17 个免费供应商的额度聚合为约 17 亿 token/月，满足个人开发者和小团队的日常需求
2. **"智能路由胜过人工配置"**：不是简单地把请求转发给第一个可用的供应商，而是用 Thompson 采样实时学习最优路由
3. **"自托管 = 隐私 + 可控"**：所有数据在本地，不存在第三方平台的数据泄露风险
4. **"一次配置，永久使用"**：OpenAI SDK 零改动，Cherry Studio / Continue / Cursor 等客户端直接使用

---

## 七、目标用户价值

| 用户类型 | 价值 |
|----------|------|
| **个人开发者** | 免费使用 100+ 模型，月省数百美元 API 费用 |
| **AI 工具开发者** | 不依赖单一供应商，自动故障转移提升可用性 |
| **隐私敏感用户** | 自托管，无第三方数据泄露风险 |
| **学生/研究者** | 零成本体验前沿模型，进行对比实验 |
| **小团队** | 统一的 API 网关，降低供应商管理成本 |

---

## 八、支持的供应商 (17个)

| 供应商 | 格式 | 主要模型 |
|--------|------|----------|
| **Google** | Gemini API | Gemini 2.5 Flash, 2.5 Pro, 3.1/3.5 Flash, Gemma 4 |
| **Groq** | OpenAI兼容 | Llama 4 Scout, GPT-OSS, Qwen3, Compound |
| **Cerebras** | OpenAI兼容 | Qwen3 235B, GLM-4.7, GPT-OSS |
| **SambaNova** | OpenAI兼容 | DeepSeek V3.1/V3.2, Llama 4 Maverick, Gemma 3 |
| **NVIDIA NIM** | OpenAI兼容 | DeepSeek V4, Nemotron 3, Kimi K2.6 |
| **Mistral** | OpenAI兼容 | Large 3, Medium 3.5, Codestral, Devstral, Magistral |
| **OpenRouter** | OpenAI兼容 | 21个免费模型 (Gemma 4, Llama, Qwen3, GLM等) |
| **GitHub Models** | OpenAI兼容 | GPT-4.1, GPT-4o |
| **Cohere** | OpenAI兼容 | Command R+, Command A |
| **Cloudflare** | OpenAI兼容 | Kimi K2.6, GLM-4.7, GPT-OSS, Granite 4 |
| **Zhipu (Z.ai)** | OpenAI兼容 | GLM-4.5, GLM-4.7 |
| **HuggingFace** | OpenAI兼容 | DeepSeek V4, Kimi K2.6, Qwen3 Coder |
| **Ollama Cloud** | OpenAI兼容 | GLM-4.7, Kimi K2, Qwen3 Coder (免费计划) |
| **Kilo Gateway** | OpenAI兼容 | Nemotron 3, Laguna (匿名免费) |
| **Pollinations** | OpenAI兼容 | GPT-OSS 20B (匿名免费) |
| **LLM7** | OpenAI兼容 | GPT-OSS, Codestral, GLM (匿名免费) |
| **OpenCode Zen** | OpenAI兼容 | DeepSeek V4, Mimo V2.5, Nemotron 3 |
| **Custom** | OpenAI兼容 | 用户自定义端点 (llama.cpp, LM Studio, vLLM等) |

---

## 九、Cherry Studio 配置指南

### 配置参数

| 配置项 | 值 |
|--------|-----|
| **API 地址 (Base URL)** | `http://localhost:3001/v1` |
| **API Key** | `freellmapi-你的统一密钥` |
| **模型** | `auto` 或指定模型如 `gemini-2.5-flash` |

### 使用步骤

1. 访问 http://localhost:5173 或 http://localhost:3001
2. 在 **Keys** 页面添加各供应商 API Key
3. 在 **Fallback Chain** 页面调整优先级
4. 复制页面顶部的 **Unified API Key**
5. 在 Cherry Studio 中配置上述参数

### 注意事项

- 不支持 Anthropic 原生格式（`/v1/messages`），仅支持 OpenAI 兼容格式
- 支持 `x-api-key` 认证头（兼容 Claude Code 等 Anthropic 客户端）
- 如果从其他机器访问，需修改 `.env` 中的 `HOST_BIND` 配置
- 每个响应返回 `X-Routed-Via` 头，显示实际使用的供应商

---

## 十、总结

FreeLLMAPI 的核心竞争力在于 **"把免费做到极致"**——不仅仅是聚合免费额度，而是通过 Thompson 采样智能路由、多维限流保护、粘性会话、自动故障转移、加密存储等一整套工程最佳实践，将"免费"提升到生产可用的水平。它在技术上不输商业 API 网关，在成本上则是零。