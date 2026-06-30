# Bun 原生提示注入分类器 — 研究计划

**状态：** P3 研究 / 早期原型
**分支：** `garrytan/prompt-injection-guard`
**骨架：** `browse/src/security-bunnative.ts`
**TODOS 锚点：** "原生 Bun 5ms DeBERTa 推断（XL, P3 / 研究）"

## 此问题解决的问题

编译后的 `browse/dist/browse` 二进制文件无法链接 `onnxruntime-node`，因为 Bun 的 `--compile` 生成一个单文件可执行文件，从临时解压目录 dlopen 依赖项，而从该目录加载原生 .dylib 失败（有文档的 oven-sh/bun#3574、#18079 + CEO 计划 §Pre-Impl Gate 1 验证）。

今天的缓解（分支 2 架构）：ML 分类器仅在 `sidebar-agent.ts`（非编译的 bun 脚本）中运行，通过 `@huggingface/transformers`。Server.ts（编译后）有零 ML —— 依赖金丝雀 + 架构控制（XML 框架 + 命令白名单）。

分支 2 的问题：分类器只能扫描 sidebar-agent 看到的内容。任何留在编译二进制文件内部的内容路径（传出的直接用户输入，仅金丝雀检查）都会错过 ML 层。

一个从头开始的 Bun 原生分类器——无原生模块，无 onnxruntime——将让编译后的二进制文件在任何地方运行完整的 ML 防御。

## 目标数字

| 指标 | 当前值（非编译 Bun 中的 WASM） | 目标值（Bun 原生） |
|---|---|---|
| 冷启动 | ~500ms（WASM 初始化） | <100mm（embeddings mmap） |
| 稳态 p50 | ~10ms | ~5ms |
| 稳态 p95 | ~30ms | ~15ms |
| 在编译二进制中工作 | 否 | 是（主要目标） |
| macOS arm64 | 可以（WASM） | 目标优先 |
| macOS x64 | 可以（WASM） | 延伸 |
| Linux amd64 | 可以（WASM） | 延伸 |

## 架构

三个构建块，按杠杆作用排序：

### 1. 分词器（完成 — 已交付 security-bunnative.ts）

纯 TS WordPiece 编码器，直接读取 HuggingFace `tokenizer.json`，为 BERT-small 词表生成与 transformers.js 相同的 `input_ids` 序列。

**为什么原生分词器本身就重要：** 分词在 transformers.js 路径中分配了大量小数组。我们的纯 TS 版本跳过了 Tensor 分配开销。温和加速（分词器本身约 5x），但更重要的是：移除了异步边界，所以冷路径以零动态导入开始。

**测试覆盖率：** `browse/test/security-bunnative.test.ts` 断言我们的 `input_ids` 在 20 个固定装置字符串上与 transformers.js 输出匹配。

### 2. 前向传播（研究 — 多周）

困难部分。BERT-small 有：
  * 12 个 transformer 层
  * 隐藏大小 512，注意力头 8
  * 总共约 30M 参数

每次前向传播是：
  1. 嵌入查找（ids → 512 维向量）
  2. 位置编码添加
  3. 12 ×（自注意力 + FFN + LayerNorm）
  4. Pooler（CLS token 投影）
  5. 分类头（2 路 sigmoid）

热路径是每 transformer 层的 12 个 matmul。每个是 ~512×512×{seq_len}。
seq_len=128 时那是 ~100 个形状为 (128, 512) @ (512, 512) 的 matmul。

**两种可行方法：**

**方法 A：纯 TS + Float32Array + SIMD**
  * 使用 Bun 的类型化数组支持 + SIMD intrinsics（当它们在 Bun 稳定版中落地时 —— 目前是仅 wasm）
  * 实现：~2000 LOC 精细的数值计算。LayerNorm、GELU、
    softmax、缩放点积注意力全部手写。
  * 延迟估计：M 系列上约 30-50ms（比使用 WebAssembly SIMD 的 WASM 明显慢）
  * **结论：** 单独不值得。纯 TS 在 matmul 方面无法击败 WASM。

**方法 B：Bun FFI + Apple Accelerate**
  * 使用 `bun:ffi` 调用 Apple 的 Accelerate 框架（cblas_sgemm）。
    在 M 系列上，cblas_sgemm 进行 768×768 matmul 约 0.5ms。
  * 权重存储为 Float32Array（在启动时从 ONNX 初始化张量加载），
    分词器在 TS 中，matmul 通过 FFI，激活在纯 TS 中。
  * 实现：~1000 LOC。数值相同，但主要工作卸载到 BLAS。
  * 延迟估计：p50 3-6ms（达到目标）。
  * RISK：仅 macOS。Linux 需要通过 FFI 的 OpenBLAS（不同的符号布局）。
    Windows 是完全独立的故事。
  * **结论：** macOS 优先的 gstack 可行。与我们现有的发布姿态
    （仅 Darwin arm64 的编译二进制）匹配。

**方法 C：Bun 中的 WebGPU**
  * Bun 在 1.1.x 中获得了 WebGPU 支持。transformers.js 已经有了
    WebGPU 后端。我们能否路由原生 Bun 通过它？
  * RISK：macOS 上的无头服务器上下文中的 WebGPU 需要适当的
    显示上下文。不确定它是否能从编译的 bun 二进制文件中工作。
  * STATUS：未探索。可能是获胜路径 —— 值得探索。

### 3. 权重加载（简单 — 已交付）

ONNX 初始化张量可以在构建时一次性提取为扁平的二进制 blob，`bun:ffi` 可以 `mmap()`。结果：运行时零解压缩。骨架还没有这样做（它通过 transformers.js 加载），但计划足够简单，一旦选择了方法 B，权重加载器是第一个要构建的东西。

## 里程碑

1. **分词器 + bench 框架**（已交付）
   分词器通过正确性测试。基准记录当前 WASM
   基线为 p50 10ms。

2. **Bun FFI 概念验证** — 来自 Apple Accelerate 的 `cblas_sgemm`，
   计时一个 768×768 matmul。确认 <1ms 延迟。

3. 单个 transformer 层的 FFI — 为 Q/K/V 投影调用 cblas_sgemm，
   在 TS 中实现 LayerNorm + softmax。在相同 input_ids 上与 onnxruntime 比较输出。
   必须在 1e-4 绝对误差内匹配。

4. 完整前向传播 — 连接所有 12 层 + pooler + 分类器。
   跨 100 个固定装置字符串与 onnxruntime 比较。

5. 生产交换 — 替换 `security-bunnative.ts` 的 `classify()` 主体。
   删除 WASM 回退。

6. 量化 — 通过 Accelerate 的 cblas_sgemv_u8s8 进行 int8 matmul
   （如果可用）或回退到 onnxruntime-extensions。~50% 内存
   减少，边际速度提升。

## 为什么不在 v1 中发布？

正确性问题。预训练 transformer 的浮点重新实现是一个多周的工程努力，其中每个操作都需要与参考的 epsilon 级一致性。LayerNorm epsilon 搞错了，精度会静默漂移。softmax 溢出处理搞错了，分类器会在长输入上产生垃圾。

将其放在 P0 功能的 PR 下发布错误的风险分配。现在发布 WASM 路径（完成），证明接口（通过 `classify()` 交付），作为后续 PR 增量落地原生版本，并附带自己的正确性回归测试套件。

## 基准测试

当前基线（来自 `browse/test/security-bunnative.test.ts` 的基准模式，在 Apple M 系列上测量 —— 其他硬件可能不同）：

| 后端 | p50 | p95 | p99 | 备注 |
|---|---|---|---|---:|
| transformers.js（WASM） | ~10ms | ~30ms | ~80ms | 预热后 |
| bun-native（存根 — 委托） | 与 WASM 相同 | | | 按设计匹配 |

当方法 B（Accelerate FFI）落地时，这一行会用新数字刷新并在提交消息中标记增量。
