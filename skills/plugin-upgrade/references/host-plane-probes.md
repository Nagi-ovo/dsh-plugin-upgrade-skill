# 宿主平面的双 cohort 探测（`cordis.patch.yml` 里的 `!!js`）

承接 [rollup R-02](rollup-0.1.2.md)。宿主平面插件（TUI 一类）不改产物，把 cohort 差异放进组合层的 `!!js` 探测。三种形态，取自 [dsh-TUI #622](https://github.com/ccch1mneyyy/dsh-TUI/pull/622) 的真实 patch：

```yaml
# 1. 新宿主才有的子路径：resolve 得到才插入，否则整行自禁用；官方同名行启用时也让位
- id: dsh-tui-subagent-model-selection-settings
  name: '@deepseek-ai/dsh-tool-subagent/model-selection-settings'
  disabled: !!js >-
    (() => {
      const require = process.getBuiltinModule('node:module').createRequire(ctx.baseUrl)
      try { require.resolve('@deepseek-ai/dsh-tool-subagent/model-selection-settings') } catch { return true }
      return [...ctx.loader.entries()].some(entry => entry.options.id === 'subagent-model-selection-settings' && !entry.disabled)
    })()

# 2. 能力被 shipped preset 接管：读目标 tag 真实存在的 preset 文件判断，宿主行让位
- id: command-goal
  disabled: !!js >-
    (() => {
      const require = process.getBuiltinModule('node:module').createRequire(ctx.baseUrl)
      const fs = process.getBuiltinModule('node:fs')
      try {
        const preset = require.resolve('@deepseek-ai/dsh-agent-presets/presets/standard/agent.cordis.yml')
        return fs.readFileSync(preset, 'utf8').split(/\r?\n/u).some(line => line.trim() === '- id: command-goal')
      } catch { return false }
    })()

# 3. 配置形态随 cohort 变：探测包目录决定给不给 roots（见 DSH-0.1.2-A1-21）
- id: dsh-tui-agent-presets
  name: '@deepseek-ai/dsh-agent-presets'
  config: !!js >-
    (() => {
      const require = process.getBuiltinModule('node:module').createRequire(ctx.baseUrl)
      const fs = process.getBuiltinModule('node:fs'), path = process.getBuiltinModule('node:path')
      try {
        const manifest = require.resolve('@deepseek-ai/dsh-agent-presets/package.json')
        if (fs.existsSync(path.join(path.dirname(manifest), 'presets'))) return { default: 'standard' }
      } catch {}
      return { default: 'standard', roots: [{ path: /* rc.2 的 dsh CLI config/agent-presets 目录 */ '', trust: 'system' }] }
    })()
```
两条纪律：

1. 探测目标必须是目标 tag 真实存在的东西——`require.resolve` 子路径、包内文件、包目录。拿别的 loader 行当"能力标记"会在那一行被改动时静默失效（dsh-TUI 第一版用 `plugin-package-inventory-deepseek` 代表 alpha，review 后改掉）。
2. 探测结果进快照：`!!js` 表达式在 rc.2 和 alpha 两个基线上分别求值后比对（dsh-TUI 的 `verify:patch-surface` 用 `@deepseek-ai/cordis-plugin-loader` 的 `evaluate` 按 `baseUrl` 求值，快照记有效状态而不是 YAML 原文）。
