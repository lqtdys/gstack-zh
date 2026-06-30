# 向 gstack 添加新宿主

gstack 使用声明式宿主配置系统。每个支持的 AI 编码代理
（Claude、Codex、Factory、Kiro、OpenCode、Slate、Cursor、OpenClaw）
都定义为一个类型化的 TypeScript 配置对象。添加新宿主意味着创建一个文件
并重新导出它。生成器、设置脚本或工具无需任何代码更改。

## 工作原理

```
hosts/
├── claude.ts        # 主宿主
├── codex.ts         # OpenAI Codex CLI
├── factory.ts       # Factory Droid
├── kiro.ts          # Amazon Kiro
├── opencode.ts      # OpenCode
├── slate.ts         # Slate (Random Labs)
├── cursor.ts        # Cursor
├── openclaw.ts      # OpenClaw（混合模式：config + adapter）
└── index.ts         # 注册表：导入所有配置，派生 Host 类型
```

每个配置文件导出一个 `HostConfig` 对象，告诉生成器：
- 在何处放置生成的技能（路径）
- 如何转换 frontmatter（白名单/黑名单字段）
- 需要重写哪些 Claude 特定引用（路径、工具名称）
- 自动安装时检测哪个二进制文件
- 抑制哪些 resolver 区段
- 安装时创建哪些符号链接资产

生成器、设置脚本、平台检测、卸载、健康检查、worktree 复制和测试都从这些配置中读取。它们都不包含针对特定宿主的代码。

## 分步指南：添加新宿主

### 1. 创建配置文件

复制现有配置作为起点。`hosts/opencode.ts` 是一个很好的最小示例。`hosts/factory.ts` 展示了工具重写和条件字段。
`hosts/openclaw.ts` 展示了适用于具有不同工具模型的宿主的适配器模式。

创建 `hosts/myhost.ts`：

```typescript
import type { HostConfig } from '../scripts/host-config';

const myhost: HostConfig = {
  name: 'myhost',
  displayName: 'MyHost',
  cliCommand: 'myhost',        // 用于 `command -v` 检测的二进制文件名
  cliAliases: [],              // 备选二进制文件名

  globalRoot: '.myhost/skills/gstack',
  localSkillRoot: '.myhost/skills/gstack',
  hostSubdir: '.myhost',
  usesEnvVars: true,           // 仅 Claude 为 false（使用字面量 ~ 路径）

  frontmatter: {
    mode: 'allowlist',         // 'allowlist' 仅保留列出的字段
    keepFields: ['name', 'description'],
    descriptionLimit: null,    // 对有长度限制的宿主设为 1024
  },

  generation: {
    generateMetadata: false,   // 仅 Codex 为 true（openai.yaml）
    skipSkills: ['codex'],     // codex 技能仅 Claude 使用
  },

  pathRewrites: [
    { from: '~/.claude/skills/gstack', to: '~/.myhost/skills/gstack' },
    { from: '.claude/skills/gstack', to: '.myhost/skills/gstack' },
    { from: '.claude/skills', to: '.myhost/skills' },
  ],

  runtimeRoot: {
    globalSymlinks: ['bin', 'browse/dist', 'browse/bin', 'gstack-upgrade', 'ETHOS.md'],
    globalFiles: { 'review': ['checklist.md', 'TODOS-format.md'] },
  },

  install: {
    prefixable: false,
    linkingStrategy: 'symlink-generated',
  },

  learningsMode: 'basic',
};

export default myhost;
```

### 2. 在索引中注册

编辑 `hosts/index.ts`：

```typescript
import myhost from './myhost';

// 添加到 ALL_HOST_CONFIGS 数组：
export const ALL_HOST_CONFIGS: HostConfig[] = [
  claude, codex, factory, kiro, opencode, slate, cursor, openclaw, myhost
];

// 添加到重新导出：
export { claude, codex, factory, kiro, opencode, slate, cursor, openclaw, myhost };
```

### 3. 添加到 .gitignore

将 `.myhost/` 添加到 `.gitignore`（生成的技能文档已被 git 忽略）。

### 4. 生成并验证

```bash
# 为新宿主生成技能文档
bun run gen:skill-docs --host myhost

# 验证输出存在且无 .claude/skills 泄漏
ls .myhost/skills/gstack-*/SKILL.md
grep -r ".claude/skills" .myhost/skills/ | head -5
# （应为空）

# 为所有宿主生成（包含新宿主）
bun run gen:skill-docs --host all

# 健康面板显示新宿主
bun run skill:check
```

### 5. 运行测试

```bash
bun test test/gen-skill-docs.test.ts
bun test test/host-config.test.ts
```

参数化冒烟测试会自动发现新宿主。无需编写测试代码。它们验证：输出存在、无路径泄漏、有效的 frontmatter、新鲜度检查通过、codex 技能已排除。

### 6. 更新 README.md

在相应部分添加新宿主的安装说明。

## 配置字段参考

参见 `scripts/host-config.ts` 了解完整的 `HostConfig` 接口，每个字段都有 JSDoc 注释。

关键字段：

| 字段 | 用途 |
|-------|---------|
| `frontmatter.mode` | `allowlist`（仅保留列出的）或 `denylist`（去除列出的） |
| `frontmatter.descriptionLimit` | 最大字符数，`null` 表示无限制 |
| `frontmatter.descriptionLimitBehavior` | `error`（中断构建）、`truncate`、`warn` |
| `frontmatter.conditionalFields` | 基于模板值添加字段（例如 sensitive → disable-model-invocation） |
| `frontmatter.renameFields` | 重命名模板字段（例如 voice-triggers → triggers） |
| `pathRewrites` | 内容的字面 replaceAll。顺序很重要。 |
| `toolRewrites` | 重写 Claude 工具名称（例如 "use the Bash tool" → "run this command"） |
| `suppressedResolvers` | 对该宿主返回空的 resolver 函数 |
| `coAuthorTrailer` | Git co-author 字符串 |
| `boundaryInstruction` | 跨模型调用的反注入警告 |
| `adapter` | 复杂转换的适配器模块路径 |

## 适配器模式（适用于具有不同工具模型的宿主）

如果字符串替换工具重写不够用（宿主有根本不同的工具语义），使用适配器模式。参见 `hosts/openclaw.ts`
和 `scripts/host-adapters/openclaw-adapter.ts`。

适配器作为所有通用重写之后的后续处理步骤运行。
它导出 `transform(content: string, config: HostConfig): string`。

## 验证

`scripts/host-config.ts` 中的 `validateHostConfig()` 函数检查：
- 名称：小写字母数字加连字符
- CLI 命令：字母数字加连字符/下划线
- 路径：仅安全字符（字母数字、`.`、`/`、`$`、`{}`、`~`、`-`、`_`）
- 配置间无重复名称、hostSubdir 或 globalRoot

运行 `bun run scripts/host-config-export.ts validate` 检查所有配置。
