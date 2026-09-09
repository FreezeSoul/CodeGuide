---
title: 第3-4节：Excel工具与XLSX渲染
pay: https://t.zsxq.com/RPzTM
---

# 《WaLiOffice - AI Agent 智能办公平台》第3-4节：Excel工具与XLSX渲染

作者：小傅哥
<br/>博客：[https://bugstack.cn](https://bugstack.cn)

>沉淀、分享、成长，让自己和他人都能有所收获！😄

大家好，我是技术UP主小傅哥。

前两节的 `md_generate` 和 `doc_generate` 输出的是“文档”。这节我们处理办公场景里另一类高频需求——**数据表格**：需求池、排期表、渠道效果分析、商机漏斗、预算表。这些内容用文字写不直观，用户最终要的是一份**能筛选、能排序、能继续编辑的 Excel**。

工具本体叫 `sheet_generate`，设计思路与 doc_generate 一脉相承：**LLM 只负责内容与结构，格式与文件交给确定性的代码**。但和 Word 不同，本分支上 Excel 的“渲染层”走了一条类似 PPT 的路：`xlsx_render.rs的 stub，真实的表格展示与编辑发生在前端在线表格组件，导出 XLSX 走 `/api/excel/export` 端点。

> **代码状态说明**：`ch03-04-sheet-generate` 提交里 `sheet_generate` 工具完整实现（场景推断、Prompt 构建、LLM 调用、JSON 解析、产物封装）；`server/src/render/xlsx_render.rs` 保留文件内标注“完整实现在 3-4 节”，当前输出空文件），`/api/excel/export` 端点已接好。也就是说：**数据结构、LLM、产物封装、自动落盘尝试、前端表格与导出按钮都已就绪**，只是“后端把 JSON 转成真 xlsx”那一步留给本节讲解并动手实现。阅读时注意区分“工具已落地的部分”和“渲染器讲解部分”，不要把设计稿当成已提交代码。

## 一、本章诉求

1. **理解多表 Schema 设计**：`SheetOutput { title, tables[], summary }`、`SheetTable { title, headers, rows, summary }` 的字段设计，以及 `#[serde(default)]` 的容错意义
2. **实现场景推断**：`infer_sheet_scene` 用关键词匹配六类业务场景，给 LLM 注入“该设计哪些列”的领域知识
3. **掌握数据真实性 Prompt 约束**：为什么必须显式禁止“示例1/示例2”式占位数据
4. **掌握 JSON 容错解析**：`extract_json` 的三级降级（去围栏 → 首尾大括号截取 → 数组截取）
5. **理解产物双链路**：`kind: "sheet"` 产物如何被 Chat 路由自动落盘（stub 渲染），用户点按钮时又如何走 `/api/excel/export` 手动导出
6. **理解前端在线表格**：多表 Tab 切换、单元格可编辑、`onUpdate` 回写 artifact，导出时用的是**编辑后的最新数据**

## 二、流程设计

### 2.1 sheet_generate 全链路

<div align="center">
    <img src="https://bugstack.cn/images/article/project/walioffice/walioffice-3-4-01.png" width="950px">
</div>

```
用户需求（topic）
    ↓
① call() 提取参数：topic 必填、scene_guide = infer_sheet_scene(topic)
        ↓
② ctx.send("state_update")：正在生成《{topic}》表格...
        ↓
③ 构建 Prompt：system（严格 JSON 格式 + 数据真实性约束）
              + user（场景偏好 + 用户需求）
        ↓
④ LlmClient::for_user().chat() → LLM 返回 JSON 文本
        ↓
⑤ extract_json 三级容错解析 → serde 反序列化为 SheetOutput
        ↓
⑥ 封装 ToolArtifact { kind: "sheet" }
   content = { type, title, tables }
        ↓
⑦ ToolResult::ok（observation 带“共 N 个表格、M 行数据”）
        ↓
⑧ AgentEvent::Artifact → SSE →_artifact_to_files
   kind == "sheet" → 取 content.tables → render_xlsx（stub：空文件）
                     → save_file_bytes 存入文件系统/库
        ↓
前端右侧面板：SheetArtifact 在线表格（Tab 切换 + 单元格编辑）
用户点“导出 XLSX” → POST /api/excel/export → render_xlsx → 下载 .xlsx
```
