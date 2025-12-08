# 工具调用无限循环问题修复总结

## 问题现象

用户在使用工具调用（Tool Use）功能时，AI 会重复调用相同的工具，进入无限循环：

```
用户: 帮我看一下 index.html 是否提交
AI: 好的我来帮你检查
*tool use: git diff
*tool use: git status
AI: 用户说xxxx，好的我来帮你检查  ← 重复！
*tool use: git diff
*tool use: git status
AI: 用户说xxxx，我马上来检查  ← 又重复！
... 无限循环 ...
```

## 根本原因

**代码位置**：`claude_converter.py` 的 `process_history()` 函数（第290-320行）

**问题**：合并连续USER消息时，没有区分包含 `tool_result` 的消息和普通文本消息，导致它们被错误地合并在一起。

**具体场景**：
```python
# 输入消息
[
  USER: "M: 检查文件",
  ASSISTANT: [tool_use...],
  USER: [tool_result...],     # 包含工具执行结果
  USER: "用户的跟进问题",     # 普通文本（连续的USER消息）
]

# 旧代码输出（错误）
[
  USER: "M: 检查文件",
  ASSISTANT: [tool_use...],
  USER: "用户的跟进问题" + [tool_result...]  # ❌ 被合并了！
]
```

**后果**：
- AI 无法识别已执行的工具调用
- 消息历史不完整
- AI 认为需要重新执行工具
- 进入无限循环

## 修复方案

在合并逻辑中添加检测：

```python
# 检测是否包含 tool_result
has_tool_results = "toolResults" in user_ctx and user_ctx["toolResults"]

if has_tool_results:
    # 不合并，保持独立
    history.append(item)
else:
    # 可以合并
    pending_user_msgs.append(user_msg)
```

## 修复后效果

```python
# 相同的输入，修复后输出（正确）
[
  USER: "M: 检查文件",
  ASSISTANT: [tool_use...],
  USER: [tool_result...],        # ✅ 独立
  USER: "用户的跟进问题",        # ✅ 独立
]
```

**优势**：
- ✅ tool_result 消息保持独立
- ✅ AI 可以看到完整的工具执行历史
- ✅ 消除无限循环
- ✅ 提高对话质量

## 测试验证

所有测试通过：
- ✅ 标准工具调用流程（3条消息）
- ✅ 多轮工具调用（5条消息，2轮对话）
- ✅ 连续USER消息（包含tool_result + 普通文本）
- ✅ 现有功能不受影响

代码质量：
- ✅ 代码审查通过
- ✅ 安全检查通过（CodeQL 0 alerts）

## 其他改进

1. **消息顺序验证**：新增 `_validate_message_order()` 函数
2. **增强循环检测**：改进 `_detect_tool_call_loop()` 函数
3. **调试模式**：环境变量 `DEBUG_MESSAGE_CONVERSION=true`
4. **完善文档**：[详细修复文档](FIX_INFINITE_LOOP_CN.md)

## 如何使用

### 升级

1. 拉取最新代码
2. 重启服务
3. 问题自动解决

### 调试（可选）

如果需要查看详细日志：

```bash
# 在 .env 中添加
DEBUG_MESSAGE_CONVERSION=true

# 重启服务后会看到详细的消息转换日志
```

## 相关资源

- [详细修复文档](FIX_INFINITE_LOOP_CN.md)
- [README 故障排查章节](../README.md#故障排查)
- [GitHub Issue 讨论](https://github.com/CassiopeiaCode/q2api/issues)

---

**修复版本**: v1.0  
**修复日期**: 2025-12-08  
**影响范围**: 所有使用工具调用的场景
