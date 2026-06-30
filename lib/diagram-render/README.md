# diagram-render

make-pdf 和 /diagram 的离线图表渲染。一个自包含的 HTML 页面（`dist/diagram-render.html`，约 9MB）捆绑了 mermaid、excalidraw 导出工具以及官方的 mermaid→excalidraw 转换器。browse daemon 通过 `load-html` 加载它；调用者通过 `browse js` 驱动它，并通过 `js --out` 拉回字节。

构建的页面是**提交的**（eng-review D2）：在零网络时即可完成安装和渲染，且 `./setup` 中没有 npm 供应链表面。偏移测试（`test/diagram-render-drift.test.ts`）在 `dist/` 被手动编辑或与 `BUILD_INFO.json` 不同步时会让 CI 失败。

## 页面 API（window 函数）

| 函数 | 输入 → 输出 |
|---|---|
| `__renderMermaid(id, text)` | mermaid 文本 → SVG 字符串。`id` 在每个围栏中唯一（`mermaid-fence-<n>`）——它对每个内部 SVG id 命名空间。 |
| `__mermaidToExcalidraw(text)` | mermaid 文本 → `.excalidraw` 场景 JSON（flowchart 完全支持；其他类型降级到上游）。 |
| `__excalidrawToSvg(sceneJson)` | 场景 JSON → SVG 字符串（嵌入了 Excalifont，离线可用）。 |
| `__rasterize(svg, targetWidthPx)` | SVG → PNG 数据 URL。调用者负责 DPI 计算：`targetWidthPx = 放置宽度（英寸）× 300`。在污染的 canvas 上抛出错误。 |
| `__downscaleRaster(dataUri, targetWidthPx, mime)` | 栅格数据 URI → 较小数据 URI（相同 mime，`targetWidthPx`）。make-pdf 用它将超尺寸照片标准化为打印分辨率。 |
| `__mountForScreenshot(svg, px)` | 防污染回退：在 `#raster-stage` 挂载 SVG 以供 `browse screenshot --selector` 使用。 |
| `__probeImage(src)` | 数据 URI/URL → `{width, height}` JSON。 |
| `__bundleInfo` | `{ name, deps }` —— 构建时固定的依赖版本。 |

就绪状态：轮询直到 `#status` 文本为 `ready`（或 `browse wait '#done'`）。页面错误累积在 `window.__errors` 中。

## 更新

```bash
# 1. 编辑 package.json 中的精确版本钉
cd lib/diagram-render && bun install
# 2. 重建（确定性的；构建两次 → 相同的 sha）
bun run build
# 3. 一起提交 package.json + bun.lock + dist/
```

渲染合约详情（securityLevel strict、htmlLabels false、print-css 字体锁定、`<base href>` + `</scri` 转义）记录在 `src/entry.ts` 和 `scripts/build.ts` 中——修改任一文件前请先阅读两者。
