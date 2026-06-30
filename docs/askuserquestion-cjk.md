# AskUserQuestion — 非 ASCII / CJK 字符

按需阅读此文档，当 AskUserQuestion 包含中文（繁體/簡體）、日文、韩文或其他非 ASCII 文本时。操作规则位于始终加载的 AskUserQuestion 自检（"Non-ASCII characters written directly, NOT \\u-escaped"）；此文档是完整的原理说明。

## 规则

当任何字符串字段（问题、选项标签、选项描述）含有非 ASCII 文本时，在 JSON 字符串中直接发出字面 UTF-8 字符。**永远不要将它们转义为 `\\uXXXX`。**

Claude Code 的工具参数管道原生支持 UTF-8，字符原封不动传递。仅保留 JSON 必需的转义：`\\n`、`\\t`、`\\"`、`\\\\`。

## 为什么转义会失败

手动转义需要从训练中回忆每个码点对长 CJK 字符串来说不可靠 —— 模型经常发出错误的码点。例如：写 `㄃` 以为是 管（U+7BA1），但 `㄃` 实际上是 ㄃，所以用户看到的是 `管理工具` 渲染为 `㄃3用箱`。

触发条件是长的多行问题包含数百个 CJK 字符：正是反射性转义激活的时候，正是错误编码最具破坏性的地方。长 ≠ 转义。保持字面值。

- 错误：`"question": "請選擇\\uXXXX\\uXXXX\\uXXXX\\uXXXX"`
- 正确：`"question": "請選擇管理工具"`
