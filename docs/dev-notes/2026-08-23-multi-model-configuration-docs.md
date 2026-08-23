# 多模型配置文档

**日期**: 2026-08-23
**类型**: docs
**分支**: dev/docs-multi-model-config
**状态**: ✅ 已完成

## 目标

在 README 中说明如何同时配置 GPT-5.6、DeepSeek Chat 和 GLM Chat，并从 Codex 会话中选择目标模型。

## 验收标准

- [x] 列出三个模型的 Model ID、上游模型、API 格式和 Base URL
- [x] 说明 GUI 中保存模型、写入客户端配置和启动代理的操作顺序
- [x] 提供 Codex 按会话选择模型的命令示例
- [x] 文档字段和按钮名称与当前实现一致

## 影响范围

- 涉及文件：`README.md`, `docs/CHANGELOG.md`
- 风险评估：低；仅补充用户文档，不改变程序行为

## 实施记录

### Step 1: 补充多模型配置说明

- 改动：新增模型参数表、GUI 配置步骤和 `codex-uni -m` 示例
- 验证：对照 `src/config_store.py`、`src/cli.py` 和 `src/pyqt_gui.py` 核对字段、命令与按钮名称

### Step 2: 同步变更记录

- 改动：在 CHANGELOG 的 `[Unreleased]` 下登记文档更新
- 验证：运行 Git 差异和空白错误检查

## 验证结果

| 测试项 | 结果 |
|--------|------|
| Markdown 差异检查 | ✅ |
| 实现一致性核对 | ✅ |
| 功能验证 | 不适用（纯文档改动） |
| 回归验证 | 不适用（纯文档改动） |

## 改动摘要

README 现在集中说明三个常用校园模型的并存配置及 Codex 会话切换方法，并使用当前 GUI 的实际按钮名称。

## 回滚方案

撤销本次对 README、CHANGELOG 和开发记录的文档提交。
