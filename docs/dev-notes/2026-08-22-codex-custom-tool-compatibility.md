# Codex GPT-5.6-Sol custom/freeform 工具兼容性

**日期**: 2026-08-22
**类型**: fix
**分支**: dev/fix-codex-custom-tools
**状态**: ✅ 已完成
**关联问题**: #017

## 目标

让 SHTUCodeProxy 完整传输 Codex CLI 0.146.1 为 GPT-5.6-Sol 注册的 freeform `apply_patch` 工具调用，使 Codex 能实际创建和修改文件。

## 验收标准

- [x] Codex 发出的 custom/freeform 工具定义和 `custom_tool_call_output` 历史不被代理丢弃
- [x] 流式响应按顺序包含 `response.output_item.added`、custom input delta/done、`response.output_item.done` 和 `response.completed`
- [x] custom tool 的 `call_id`、`item_id` 和 `output_index` 全程一致
- [x] 现有 `function_call` 及 `response.function_call_arguments.*` 行为不变
- [x] Codex 能创建文件、修改文件、应用多行 patch、读回内容并在工具输出后继续对话
- [x] 项目模块导入、相关回归和非 GUI smoke test 通过

## 影响范围

- 涉及文件：`src/proxy.py`, `tests/smoke_test.py`, `docs/ISSUE-TRACKER.md`, `docs/CHANGELOG.md`
- 风险评估：中；Responses 流式事件转换是 Codex 工具调用核心路径，错误的 ID 或事件顺序会同时影响已有 function tools

## 实施记录

### Step 1: 审计与复现

- 改动：登记 #017；审计 Responses 请求转换、流式解析、非流式转换和恢复分支
- 验证：使用隔离的临时 `CODEX_HOME` 和本地 8098 捕获端点确认 Codex 0.146.1 请求工具为 `{type:"custom", name:"apply_patch", format:{type:"grammar", syntax:"lark", definition:"..."}}`；标准响应为 `output_item.added → custom_tool_call_input.delta → custom_tool_call_input.done → output_item.done → response.completed`；下一轮输入包含同一 `call_id` 的 `custom_tool_call` 与 `custom_tool_call_output`

### 根因分析

`responses_request_to_upstream()` 会保留 custom tool 定义和历史，因此请求侧 Responses 直连没有丢失数据；但 `extract_text_delta()` 只识别普通 `function_call` 事件，`handle_responses_streaming()` 也只收集并重建 function calls。custom tool 事件全部落入 `ignore`，最终合成响应缺少 `custom_tool_call`，导致 Codex 在旁白后结束而不执行 patch。Chat Completions 本身没有等价 freeform 工具协议，本修复限定于 GPT-5.6-Sol 使用的 Responses 直连路径。

### Step 2: 回归测试与最小修复

- 改动：为 `extract_text_delta()` 增加 custom tool item/input 事件解析；新增独立 accumulator 和标准 SSE emitter；在 Responses 流式及非流式 fallback 中保留 custom tool call；不改普通 function call 分支
- 验证：先确认新增测试在旧实现上因 `response.output_item.added` 未识别而失败，修复后 parser、accumulator、handler 回归均通过

### Step 3: Codex 端到端验证

- 改动：无生产配置改动；使用临时 Codex home 和 8098 测试代理，先以 8099 synthetic upstream 验证确定性协议序列，再连接已有校园 GPT-5.6-Sol 配置验证真实模型响应
- 验证：Codex 0.146.1 + 校园 GPT-5.6-Sol 依次执行多行文件创建、第二次 `apply_patch` 修改、`exec_command` 读回，并在两次 `custom_tool_call_output` 和一次 `function_call_output` 后完成对话；最终文件恰为 `first\nfixed\n`

## 验证结果

| 测试项 | 结果 |
|--------|------|
| 模块导入 | ✅ Windows Python 3.13.14 |
| 冒烟测试 | ✅ 非 GUI 全量通过；完整入口因环境缺少 PyQt5 在既有 GUI 用例处停止 |
| 功能验证 | ✅ Codex 0.146.1 + 校园 GPT-5.6-Sol create → modify → read → continue（测试端口 8098） |
| 回归验证 | ✅ 普通 function call 与 custom tool handler 测试通过 |

未运行完整 5 模型 API 矩阵：当前可用运行时配置仅 GPT-5.6-Sol 有 API key，其余四个模型无 key。生产 8082 进程未停止或重启。

## 改动摘要

Responses 请求继续原样透传 custom/freeform 工具定义及历史。响应侧现在识别、累计并重新发出标准 custom tool SSE 事件，保留调用 ID 与输出序号，避免 custom-only 响应被误判为空。Chat Completions 路由没有 freeform 等价协议，因此未做不可靠转换。

## 回滚方案

恢复 `backups/src-snapshots/src-20260822-163911/` 中的源码快照，并撤销本任务对测试和文档的改动。
