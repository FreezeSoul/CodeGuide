---
title: 第3-3节：联网搜索工具 web_search
pay: https://t.zsxq.com/VFvDu
---

# 《WaLiOffice - AI Agent 智能办公平台》第3-3节：联网搜索工具 web_search

作者：小傅哥
<br/>博客：[https://bugstack.cn](https://bugstack.cn)

>沉淀、分享、成长，让自己和他人都能有所收获！😄

大家好，我是技术UP主小傅哥。

前两节的 `md_generate` 和 `doc_generate` 都是"纯生成"类工具——LLM 根据用户需求凭空创造内容。但真实办公场景里经常出现另一类需求：查竞品动态、查政策原文、查某个库的最新版本号。这些信息在 LLM 的训练数据里**不存在或者过时**，编是编不出来的。

这节我们来写第一个**不依赖 LLM 生成内容**的工具——`web_search`。它也是全项目工具里唯一一个 `is_read_only = true` 的工具：只读外部世界，不产生新内容，产物是一张"搜索结果卡片"。

但联网搜索绝不是"调一个搜索 API"就完事。真实工程里要解决四个问题：

1. **多 Provider**：中文搜索和英文搜索的最优数据源不同，需要一个 Provider 抽象
2. **自动降级**：搜索服务是最不稳定的外部依赖（反爬、限流、Key 过期），主源挂了要无缝切换
3. **协议适配**：百度 MCP 走 SSE + JSON-RPC，SearXNG 返回 JSON，百度移动版/DuckDuckGo 返回 HTML——三种协议要统一成一个返回结构
4. **链路可观测**：降级发生后，用户应该能看到"这次到底用了哪个源"

## 一、本章诉求

1. **理解 SearchProvider 抽象**：四种 Provider 的数据源、协议和适用场景
2. **掌握 auto 模式的语言路由**：`contains_cjk` 判断中英文，动态组装降级链，未配置 Key 的 Provider 自动剔除
3. **实现百度 MCP 客户端**：SSE 长连接 + JSON-RPC 握手（initialize → notifications/initialized → tools/call），流式等待结果
4. **掌握三种结果解析**：JSON 直接反序列化（SearXNG）、HTML 正则提取（百度/DuckDuckGo）、纯文本正则提取（百度 MCP）
5. **理解结果清洗链**：HTML 标签清理、HTML 实体转义、URL 归一化（DuckDuckGo 的 `uddg` 重定向参数）、snippet 截断
6. **掌握产物与状态设计**：`kind: "search"` 产物、两次 `state_update`（搜索中 → 检索来源）、`providers_tried` 链路透出

## 二、流程设计

### 2.1 web_search 调用链路

<div align="center">
    <img src="https://bugstack.cn/images/article/project/walioffice/walioffice-3-3-01.png" width="950px">
</div>

```
ReAct 循环：LLM 决策调用 web_search(query, max_results)
        ↓
① 参数提取：query trim、max_results clamp(1, 10)
   query 为空 → ToolResult::err("query 不能为空")
        ↓
② ctx.send("state_update")：正在搜索：{query}
        ↓
③ search_web(query, max_results)
   ├─ 读配置 AIPPT_WEB_SEARCH_PROVIDER（默认 auto）
   ├─ 构建共享 reqwest Client（超时 AIPPT_WEB_SEARCH_TIMEOUT_MS，默认 20s）
   ├─ 指定 provider → 直接调用，失败即失败
   └─ auto → contains_cjk(query) 判断语言
        ↓ 中：[baidu_mcp(有Key才加), baidu, searxng, duckduckgo]
        ↓ 英：[searxng, duckduckgo, baidu_mcp(有Key才加), baidu]
        ↓ 依次尝试，第一个返回非空结果的 Provider 胜出
        ↓
④ ctx.send("state_update")：本次使用 {provider_label}，检索链路：xxx -> yyy
        ↓
⑤ 构建 ToolArtifact { kind: "search" }
   content = { type, query, provider, provider_label, providers_tried, results }
        ↓
⑥ ToolResult
   ├─ observation：结果列表编号文本（给 LLM 供后续引用）
   └─ data：同产物 JSON（给 Agent/前端参考）
        ↓
前端 SearchArtifact 渲染：查询词、来源、结果数量、检索链路标签、结果卡片列表
```

