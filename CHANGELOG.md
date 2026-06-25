# 更新日志 (Changelog)

本项目的所有主要更改和版本迭代说明将记录于此。

## [1.1.0] - 2026-06-25

### 🌟 新增特性 (New Features)
- **突破性更新：支持全模态输入 (Multimodal Input Support)**
  - 成功攻克网页端代理此前受限于 WIZ RPC 协议无法发送图片和文件的致命限制。
  - **混合 API 视觉降维技术**：通过引入全新的 `official_api_key` 配置项，允许代理静默调用官方免费的高速 `gemini-1.5-flash` API 充当“前置视觉解析层”。
  - **自动嗅探与智能拦截**：在底层自动拦截带有 `image_url` 或 `file_url` 的 OpenAI 标准 Payload 请求。
  - **全格式覆盖**：不仅支持纯图片，还能完美解析和转写 **PDF 文档** 与 **音频文件 (Audio)** 的 Base64 数据流。
  - **无缝拼装**：将提取出的高质量画面描述、OCR 文字、语音转写结果无缝对接到纯文本 Prompt 中，交付给网页版长文本模型（如 Flash-Thinking）进行深度推演。

### 🛠️ 核心优化 (Core Optimizations)
- **Codex 客户端深度适配**：补全了残缺的 `response.output_item.added` 和 `response.content_part.added` 等 SSE 服务器推送事件，彻底修复了 Codex 及部分主流第三方客户端在 Pure API 模式下气泡渲染失败、打字机效果卡顿的 Bug。
- **配置项扩展**：`config.json` 原生支持配置 `official_api_key` 字段。
- 在 `README.md` 与 `README_CN.md` 中同步更新了多模态功能的详细介绍和最新配置指南。

### 🐞 错误修复 (Bug Fixes)
- 修复了因为非标准返回导致部分客户端抛出 401 Unauthorized 异常的问题。
- 追加忽略了运行过程中产生的冗余备份文件 `.working_backup` 避免污染主分支。

---

## [1.0.0] - 初始版本 (Initial Fork)

- 基于原作者仓库分叉构建的基础反向代理功能。
- 提供将 Google Gemini 网页端逆向转换为标准 OpenAI API ( `/v1/chat/completions` ) 的能力。
- 支持单文件跨平台部署。
