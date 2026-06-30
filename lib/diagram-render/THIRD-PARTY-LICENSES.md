# 第三方许可证 — diagram-render 打包文件

`dist/diagram-render.html` 捆绑了以下包（精确版本钉在 `package.json` 中；通过 `bun.lock` 解析传递依赖）：

| 包 | 版本 | 许可证 | 来源 |
|---|---|---|---|
| mermaid | 11.12.2 | MIT | https://github.com/mermaid-js/mermaid |
| @excalidraw/excalidraw | 0.18.0 | MIT | https://github.com/excalidraw/excalidraw |
| @excalidraw/mermaid-to-excalidraw | 1.1.2 | MIT | https://github.com/excalidraw/mermaid-to-excalidraw |
| react | 18.3.1 | MIT | https://github.com/facebook/react |
| react-dom | 18.3.1 | MIT | https://github.com/facebook/react |

该包还嵌入了 @excalidraw/excalidraw 附带的字体（Excalifont 及相关字族），根据 excalidraw 仓库许可为 SIL Open Font License 1.1。

更新版本钉时，请重新验证其许可证字段（`bun pm ls` 或包的 LICENSE 文件）并在同一提交中更新此表。
