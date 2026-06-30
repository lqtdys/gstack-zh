# 设计系统 — gstack

## 产品背景
- **这是什么：** gstack 的社区网站 — 一个将 Claude Code 变成虚拟工程团队的 CLI 工具
- **目标用户：** 发现 gstack 的开发者、现有社区成员
- **领域/行业：** 开发者工具（同类：Linear、Raycast、Warp、Zed）
- **项目类型：** 社区仪表盘 + 营销网站

## 美学方向
- **方向：** 工业/实用主义 — 功能优先、数据密集、等宽字体作为个性字体
- **装饰程度：** 刻意 — 表面微妙的噪点/颗粒纹理营造质感
- **氛围：** 一个用心打造的严肃工具。温暖，而非冷漠。CLI 传承本身就是品牌。
- **参考站点：** formulae.brew.sh（竞品，但我们的是实时交互的）、Linear（暗色 + 克制）、Warp（暖色点缀）

## 字体排印
- **展示/标题：** Satoshi（Black 900 / Bold 700）— 几何感带温暖，独特的字母形式（小写 'a' 和 'g'）。不是 Inter，不是 Geist。从 Fontshare CDN 加载。
- **正文：** DM Sans（Regular 400 / Medium 500 / Semibold 600）— 干净、可读，比几何展示字体稍友好。从 Google Fonts 加载。
- **UI/标签：** DM Sans（与正文相同）
- **数据/表格：** JetBrains Mono（Regular 400 / Medium 500）— 个性字体。支持 tabular-nums。等宽不应只在代码块中出现。从 Google Fonts 加载。
- **代码：** JetBrains Mono
- **加载：** Google Fonts 用于 DM Sans + JetBrains Mono，Fontshare 用于 Satoshi。使用 `display=swap`。
- **字号比例：**
  - Hero: 72px / clamp(40px, 6vw, 72px)
  - H1: 48px
  - H2: 32px
  - H3: 24px
  - H4: 18px
  - 正文: 16px
  - 小: 14px
  - 说明: 13px
  - 微小: 12px
  - 极小: 11px（JetBrains Mono 标签）

## 颜色
- **策略：** 克制 — 琥珀色点缀是稀有的、有意义的。仪表盘数据才有色；界面框架保持中性。
- **主色（暗色模式）：** amber-500 #F59E0B — 温暖、有活力，被理解为"终端光标"
- **主色（亮色模式）：** amber-600 #D97706 — 在白色背景上有更好的对比度
- **主文本色（暗色模式）：** amber-400 #FBBF24
- **主文本色（亮色模式）：** amber-700 #B45309
- **中性色：** 冷锌灰
  - zinc-50: #FAFAFA（最浅）
  - zinc-400: #A1A1AA
  - zinc-600: #52525B
  - zinc-800: #27272A
  - 表面（暗色）：#141414
  - 基底（暗色）：#0C0C0C
  - 表面（亮色）：#FFFFFF
  - 基底（亮色）：#FAFAF9
- **语义色：** success #22C55E, warning #F59E0B, error #EF4444, info #3B82F6
- **暗色模式：** 默认。近黑基底（#0C0C0C），表面卡片为 #141414，边框为 #262626。
- **亮色模式：** 暖石基底（#FAFAF9），白色表面卡片，石色边框（#E7E5E4）。琥珀色点缀变为 amber-600 以增强对比。

## 间距
- **基准单位：** 4px
- **密度：** 舒适 — 不紧凑（不是彭博终端），不宽松（不是营销网站）
- **比例：** 2xs(2px) xs(4px) sm(8px) md(16px) lg(24px) xl(32px) 2xl(48px) 3xl(64px)

## 布局
- **策略：** 仪表盘用栅格约束，着陆页用编辑推荐
- **栅格：** lg+ 12列，移动端 1列
- **最大内容宽度：** 1200px（6xl）
- **边框半径：** sm:4px, md:8px, lg:12px, full:9999px
  - 卡片/面板：lg（12px）
  - 按钮/输入框：md（8px）
  - 徽章/药丸：full（9999px）
  - 技能条：sm（4px）

## 动效
- **策略：** 最小化功能 — 仅帮助理解的过渡。仪表盘的实时信息流就是动效。
- **缓动：** enter(ease-out / cubic-bezier(0.16,1,0.3,1)) exit(ease-in) move(ease-in-out)
- **时长：** micro(50-100ms) short(150ms) medium(250ms) long(400ms)
- **动画元素：** 实时信息流点脉冲（2s无限循环）、技能条填充（600ms ease-out）、悬停状态（150ms）

## 颗粒纹理
为整个页面添加微妙的噪点叠加以增加质感：
- 暗色模式：不透明度 0.03
- 亮色模式：不透明度 0.02
- 使用 SVG feTurbulence 滤镜作为 body::after 的 CSS 背景图像
- pointer-events: none, position: fixed, z-index: 9999

## 决策日志

| 日期 | 决策 | 理由 |
|------|------|------|
| 2026-03-21 | 初始设计系统 | 由 /design-consultation 创建。工业美学、暖琥珀色点缀、Satoshi + DM Sans + JetBrains Mono。 |
| 2026-03-21 | 亮色模式 amber-600 | amber-500 在白色上太亮/发白；amber-700 太棕/赭色。amber-600 是最佳点。 |
| 2026-03-21 | 颗粒纹理 | 为平面暗色表面增添质感。防止"通用 SaaS 模板"的同质化。 |
