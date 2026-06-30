# 计划：快照下拉/自动完成交互元素检测

## 问题

`snapshot -i` 在现代 Web 应用中遗漏了下拉/自动完成项。这些元素：
1. 通常是无语义 ARIA 角色的带点击事件的 `<div>`/`<li>`
2. 驻留在动态创建的 portal/popover 中（浮动容器）
3. 不出现在 Playwright 的可访问性树（`ariaSnapshot()`）中

`-C` 标志（光标交互扫描）为此设计但存在以下问题：
- 需要单独使用标志 — 使用 `-i` 的代理不会自动获得它
- 跳过有 ARIA 角色的元素（即使 ARIA 树遗漏了它们）
- 不优先处理下拉项所在的 popover/portal 容器

## 根因

Playwright 的 `ariaSnapshot()` 从浏览器的可访问性树构建。动态渲染的 popover（React portal、Radix Popover 等）可能不在可访问性树中，如果：
- 组件未设置 ARIA 角色
- portal 在作用域 `body` 定位器子树内但时间不匹配
- 浏览器在 DOM 突变后尚未更新可访问性树

## 变更

### 1. 使用 `-i` 标志时自动启用光标交互扫描

**文件：** `browse/src/snapshot.ts`

当传入 `-i`（交互式）时，自动包含光标交互扫描。这意味着代理在请求交互元素时始终能看到可点击的非 ARIA 元素。

`-C` 标志保持为独立选项，用于非交互式快照。

```
if (opts.interactive) {
  opts.cursorInteractive = true;
}
```

### 2. 添加 popover/portal 优先扫描

**文件：** `browse/src/snapshot.ts`（光标交互 evaluate 块内）

在通用光标:pointer 扫描之前，专门扫描可见的浮动容器（popover、下拉菜单、菜单），并将其所有直接子元素包含为交互项：

浮动容器的检测启发式：
- `position: fixed` 或 `position: absolute` 加上 `z-index >= 10`
- 有 `role="listbox"`、`role="menu"`、`role="dialog"`、`role="tooltip"`、`[data-radix-popper-content-wrapper]`、`[data-floating-ui-portal]` 等
- 在 DOM 中最近出现（不在初始页面加载）
- 可见（`offsetParent !== null` 或 `position: fixed`）

对于每个浮动容器，包含子元素需满足：
- 有文本内容
- 可见
- 有 cursor:pointer 或 onclick 或 role="option" 或 role="menuitem"
- 用 `popover-child` 原因标注

### 3. 移除光标交互扫描中的 `hasRole` 跳过

**文件：** `browse/src/snapshot.ts`

当前：`if (hasRole) continue;` — 跳过任何有 ARIA 角色的元素，假设 ARIA 树已捕获。

问题：如果 ARIA 树遗漏了该元素（时间、portal、不良 DOM 结构），它会从两个系统中掉落。

修复：仅在角色在 `INTERACTIVE_ROLES` 中且确实在主 refMap 中捕获时才跳过。否则包含。

由于无法从 `page.evaluate()` 内部轻松检查 refMap，更简单的修复：移除浮动容器内元素的 `hasRole` 跳过。对于浮动容器外部的元素，保持 `hasRole` 跳过不变（以避免正常页面内容中的重复）。

### 4. 添加下拉测试夹具和测试

**文件：** `browse/test/fixtures/dropdown.html`

HTML 包含：
- 一个组合框输入框，在焦点/输入时显示下拉
- 下拉项为带点击事件的 `<div>`（无 ARIA 角色）
- 下拉项为带 `role="option"` 的 `<li>`
- React-portal 式容器（`position: fixed`，高 z-index）

**文件：** `browse/test/snapshot.test.ts`

新测试用例：
- 在下拉页面上的 `snapshot -i` 通过光标扫描找到下拉项
- 在下拉页面上的 `snapshot -i` 包含 popover-child 元素
- 来自下拉扫描的 `@c` 引用是可点击的
- 在浮动容器内的有 ARIA 角色的元素即使被 ARIA 树遗漏也会被捕获

## 回滚风险

**低。** `-C` 扫描是叠加的 — 只添加 `@c` 引用，从不移除 `@e` 引用。自动与 `-i` 一起启用会增加输出大小，但代理已经处理混合引用类型。

**一个顾虑：** `-C` 扫描查询所有元素（`document.querySelectorAll('*')`），在沉重页面上可能较慢。对于 popover 专用扫描，我们限制在检测到的浮动容器内，这很快（小子树）。

## 测试

```bash
cd /data/gstack/browse && bun test snapshot
```

## 变更文件

1. `browse/src/snapshot.ts` — 自动启用 -C 与 -i，popover 扫描，移除浮动容器中的 hasRole 跳过
2. `browse/test/fixtures/dropdown.html` — 新测试夹具
3. `browse/test/snapshot.test.ts` — 新下拉/popover 测试用例
