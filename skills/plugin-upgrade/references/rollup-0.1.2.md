# Rollup · 0.1.1 → 0.1.2 走廊

> **状态**: 基于 `dsh-v0.1.2-alpha.2`。**0.1.2 正式版尚未发布**——npm dist-tags 实测 `latest`/`next` = `0.1.1-rc.2`，`alpha` = `0.1.2-alpha.2`。正式发版后本文件需按 final tag 复核转正（[issue #1](https://github.com/oh-my-dsh/dsh-plugin-upgrade-skill/issues/1) 的原始 caveat）。
> **定位**: 本文件不重复版本卡片。逐条变更以卡片为准，这里只写走廊层的增量——跨 cohort 共存、未发布 cohort 安装、CI/发布连带、迁移前盘点与 baseline 归因、boot race 处置、分层验证清单。
> 卡片格式见 [README.md](README.md)。触点编号对应 [pre-flight 清单](pre-flight.md)。

## 怎么用这份 rollup

0. 迁移动手前，先按 R-06 采集 baseline（即分层验证清单第 0 层）；
1. 先按 [pre-flight.md](pre-flight.md) 测出命中的触点类；
2. 按 `from → to` 读完整走廊并先计算净状态：[v0.1.2-alpha.1.md](v0.1.2-alpha.1.md)（15 张）→ [v0.1.2-alpha.2.md](v0.1.2-alpha.2.md)（6 张）；
3. 回到本文件处理走廊层问题——这些跨越单版本，卡片里没有；
4. 按本文件末尾的分层验证清单收工。

## 卡片索引（按触点）

| 触点 | 相关卡片 |
|---|---|
| #1 源码 patch | [DSH-0.1.2-A1-03](v0.1.2-alpha.1.md) |
| #2 事件 / 持久事件 | [DSH-0.1.2-A1-02](v0.1.2-alpha.1.md) → [DSH-0.1.2-A2-01](v0.1.2-alpha.2.md)，另见 [DSH-0.1.2-A1-06](v0.1.2-alpha.1.md) |
| #3 服务 / Remote | [DSH-0.1.2-A1-01](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-06](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-11](v0.1.2-alpha.1.md)、[DSH-0.1.2-A2-02](v0.1.2-alpha.2.md)、[DSH-0.1.2-A2-05](v0.1.2-alpha.2.md)、[DSH-0.1.2-A2-06](v0.1.2-alpha.2.md) |
| #4 宿主目录读写 | [DSH-0.1.2-A1-04](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-13](v0.1.2-alpha.1.md) |
| #5 UI / 命令 / 工具 | [DSH-0.1.2-A1-03](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-06](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-09](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-10](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-11](v0.1.2-alpha.1.md) |
| #6 自建 HTTP/WS/RPC/DOM/CSS | [DSH-0.1.2-A1-08](v0.1.2-alpha.1.md) |
| #7 子进程 / stdout / stderr | [DSH-0.1.2-A1-04](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-05](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-06](v0.1.2-alpha.1.md)、[DSH-0.1.2-A1-13](v0.1.2-alpha.1.md)、[DSH-0.1.2-A2-04](v0.1.2-alpha.2.md) |
| #6/#7 Web 启动与验收 | [DSH-0.1.2-A1-19](v0.1.2-alpha.1.md)（认证 URL、启动图资源发现与真实挂载） |
| 特殊面 | 权限 [DSH-0.1.2-A1-07](v0.1.2-alpha.1.md)；隐私 [DSH-0.1.2-A1-12](v0.1.2-alpha.1.md) / [DSH-0.1.2-A1-14](v0.1.2-alpha.1.md)；打包 [DSH-0.1.2-A2-03](v0.1.2-alpha.2.md) |

> **跨版本回滚型变更先读完整走廊再动手**：字段或语义在中间版本删除、后续版本又恢复
> （典型如 `ignorable` 的 [DSH-0.1.2-A1-02](v0.1.2-alpha.1.md) → [DSH-0.1.2-A2-01](v0.1.2-alpha.2.md) 一删一复）。
> 迁移时必须先折叠走廊的净状态再修改源码——不要在 alpha.1 删一次、到 alpha.2 又加回来；
> 若最终目标已恢复该语义，旧版适配里的防御代码应当删除而不是保留。

## Remote 调用的错误流

承接 [DSH-0.1.2-A1-01](v0.1.2-alpha.1.md) 与
[DSH-0.1.2-A2-02](v0.1.2-alpha.2.md)。alpha.2 的 unary Remote 返回
`Promise<RemoteResult<T>>`：业务/载体失败走 `ok: false`；参数个数、未挂载方法、缺少
Context adapter 等装配/编程错误仍可能 reject，应暴露修复而不是吞掉重试。

```typescript
const result = await ctx.remote.session.list({ limit: 10 })
if (!result.ok) {
  switch (result.error.code) {
    case 'gateway/cancelled':
      return // 结束或传播取消，不重试、不报通用错误
    case 'session/not-found':
      return null
    default:
      // 保留 code/details 并上报；仅明确瞬态、幂等且策略允许时重试
      throw result.error
  }
}
return result.value
```

只有上层接住主动 `throw result.error` 的值时，才用
`isRemoteFailure`（`@deepseek-ai/dsh-api-gateway/client`）区分 Remote failure 与本地
缺陷；本地缺陷继续抛出。禁止跨 realm 使用 `instanceof RemoteError`。`gateway/internal`
和未知码不证明请求未执行，默认保留原始 `code/details` 并上报，不盲重试。

来源：[DSH-0.1.2-A2-02](v0.1.2-alpha.2.md) 与
[ctx-remote-failure-vocabulary](https://github.com/deepseek-ai/deepseek-harness/blob/dsh-v0.1.2-alpha.2/.agents/notes/implemented/architecture/2026-08-28-ctx-remote-failure-vocabulary.md)。

## 走廊层增量

以下问题跨越单个版本或落在卡片之外，来自社区实战（[discussion #5120](https://github.com/deepseek-ai/deepseek-harness/discussions/5120)，dsh-web 约 20 个包的迁移）与迁移管线实践。

### R-01 · 目标 cohort 的依赖包未完整发布 npm

- **类型**: process
- **症状**: 根包或 dist-tag 可用不代表插件直接依赖的每个内部 cohort 包都已发布；只有实际 registry 查询返回缺失时，才进入本配方。
- **配方**: 先记录缺失的精确包名/版本。确认 registry 确实不可用后，在隔离 worktree 从官方 tag 构建并 `pnpm pack`，用 `overrides` 钉到 `file:` tarball；不要把所有 `0.1.2-alpha.*` 一概描述成 404。

  ```sh
  git clone https://github.com/deepseek-ai/deepseek-harness.git /tmp/dsh-build
  cd /tmp/dsh-build && git checkout dsh-v0.1.2-alpha.2
  pnpm install && pnpm run build
  mkdir -p ~/.dsh-cohorts/0.1.2-alpha.2
  pnpm -r exec pnpm pack --pack-destination ~/.dsh-cohorts/0.1.2-alpha.2
  ```

  manifest 里 range 写 `^0.1.2-alpha.2`，将来正式发布删掉 overrides 段即回到 registry 解析。
- **注意（待确认）**: 以下 pnpm 版本钉点来自**单一实战报告，尚未在其他仓库复现验证**——报告称 `11.9.0` 对 file: tarball 的传递依赖在有第三方 peer 时会绕过 overrides 去 registry 找不存在的版本，钉 `packageManager: pnpm@11.24.0` 才解析正确。落地前先在目标仓库做最小复现确认，验证通过后回填结果并把本条目转正（与本文件末尾「待确认」小节同步更新）。
- **验证**: `pnpm list --depth 0 | grep @deepseek-ai` 全部指向目标版本，无混合。
- **来源**: [#5120](https://github.com/deepseek-ai/deepseek-harness/discussions/5120) 第 1 条。**未在官方 release notes 覆盖范围内**，属社区实践。

### R-02 · 跨 cohort 共存（旧宿主升不到未发布 cohort）

- **类型**: breaking（组合效应）
- **症状**: rc.2 宿主不可能升到未发布的 alpha，插件必须在两个 cohort 上都跑。两条硬耦合会浮出来：客户端 bundle 求值期硬 `require` 平台模块，在没有该模块表项的宿主上报 `missed the module table`；只在新宿主注册的服务被写进硬 `inject` 清单，旧宿主上入口永久 `pending (waiting for service: …)`。
- **配方**: 一个产物 + 运行时解析 cohort 表面。

  ```typescript
  // 共享 build 预设里生成 shim：不再 externalize 值导入，求值期用注入的 require 解析
  function resolveStoreEngine() {
    // 说明符拼接构造，避免被静态 external 扫描标记
    const platform = ['@deepseek-ai', 'dsh-client-store'].join('/')
    const legacy = ['@deepseek-ai', 'dsh-client-runtime', 'client'].join('/')
    try { return require(platform) } catch { return require(legacy) }
  }
  ```

  只转**共享**的值表面——cohort 独有导出绝不能 re-export，否则新值导入会在 build 时报 missing-export 而不是静默坏掉。类型导入照旧（编译期擦除）。

  注入服务从硬 inject 清单拿掉，在使用点探测；**cordis `remote` 代理对未注入属性是 throw 而非返回 undefined**，所以必须 try/catch 再回落：

  ```typescript
  let presets
  try { presets = ctx.remote.agentPresets } catch { presets = undefined }
  const roster = presets ?? ctx.connection.api.agentPresets
  ```

- **被否决的替代**: per-consumer try/catch 重复污染源码；按宿主 cohort 出不同产物（重新引入有状态构建）；硬等 inject wait（旧宿主永久 pending）。
- **验证**: 同一份产物分别 link 到旧宿主与新宿主，各跑一次冷启动 + 完整一轮对话。
- **来源**: [#5120](https://github.com/deepseek-ai/deepseek-harness/discussions/5120) 第 3、4 条。

### R-03 · 第三方预构建插件搭不上你的 shim

- **类型**: breaking
- **症状**: 预构建 npm 内容进入 profile，硬 require 旧说明符，在新宿主同样 `missed the module table`。你不构建它，build 预设的 shim 帮不到。
- **配方**: 仓库自持 `pnpm patch`（`patchedDependencies`），把那一条 require 改写成同样的双 cohort 探测。**别忘了 profile 的父层链接**——link 脚本可能把链接重指回未打补丁实例，需指到 `patch_hash=…` 实例。
- **验证**: 打补丁后冷启动，确认该插件的 UI 贡献点可见可用。
- **来源**: [#5120](https://github.com/deepseek-ai/deepseek-harness/discussions/5120) 第 7 条。

### R-04 · CI 与发布管线连带

- **类型**: process
- **症状**: overrides 用 file: tarball 后，frozen lockfile 记录机器相关绝对路径，每个 runner `pnpm install --frozen-lockfile` 缺 store。另外 cohort 未发布期间，版本 tag 一旦触发 npm publish，会发出 `@deepseek-ai/*` range 无法从 registry 解析的包，且对该版本不可逆。
- **配方**:
  - 加一个脚本在任何机器 materialize tarball store（解析 overrides；store 已存在则秒退），用以 `pnpm-workspace.yaml`（或 `package.json`）hash 为 key 的 actions cache 服务所有 pnpm 消费 job；
  - `pnpm/action-setup` 删掉 `version` 输入，让 `packageManager` 成为唯一版本来源，避免与钉点冲突；
  - release workflow 加 `NPM_PUBLISH_ENABLED` 开关：tag 仍跑全部门禁与 smoke，跳过 npm publish，直到 cohort 正式发布。
- **验证**: 干净 runner 上 `--frozen-lockfile` 安装成功；tag 演练确认未产生 publish。
- **来源**: [#5120](https://github.com/deepseek-ai/deepseek-harness/discussions/5120) 第 8、9 条。

### R-05 · 迁移前盘点被删包的下游

- **类型**: process
- **症状**: 依赖被删 SDK 包才能构建的插件（承接 [DSH-0.1.2-A1-01](v0.1.2-alpha.1.md)），迁移中途才发现无法构建，只能随迁移退役。
- **配方**: 迁移前先对全部插件跑一次「import 了哪些被删包」的盘点，把「必须退役」与「可迁移」分开排期，而不是边迁边发现。
- **验证**: 盘点清单与实际迁移结果一致，无中途新增退役项。
- **来源**: [#5120](https://github.com/deepseek-ai/deepseek-harness/discussions/5120) 第 10 条。

### R-06 · 迁移前 baseline 归因——先立豁免清单，再动迁移

- **类型**: process
- **症状**: 插件仓库在迁移前就存在失败（旧 cohort 上测试/typecheck 已挂）。迁移后跑
  机械套件一片红，无法区分「迁移引入」与「本来就有」：顺手把预存失败修掉，会污染
  迁移 diff、掩盖真实回归；不修不报，则会把预存失败误报为迁移破坏。两个方向都真实
  发生过：干净树 + 红套件被当回归上报；脏树 + 绿门禁掩盖运行时断链（本文件分层验证
  清单针对后者，本条针对前者）。
- **配方**:
  1. 在**任何迁移写入之前**，于仓库**自身**依赖状态（不 pin 目标 cohort、不设目标
     env 变量）跑一次机械套件（build / typecheck / tests），记录失败清单与失败指纹
     ——此即 baseline。同时固定环境证据：`HEAD`、工作树状态、lockfile 哈希、解析后
     的依赖与工具版本、完整命令与退出码（时间戳只是辅助——迁移可能先改完才首次
     提交，时间戳证不了先后）。
  2. baseline 失败进入**不修豁免清单**：迁移过程绝不顺手修复预存失败——那是另一个
     PR 的事。
  3. 迁移后对比失败指纹（按命令、测试标识、规范化路径与诊断消息聚合；裸错误行只是
     近似——移行与堆栈变化会引入噪声）：只有相对 baseline **新增**的失败计入迁移
     失败；测试清单不得无依据减少（删测试让集合缩小不算变绿）。
  4. 修复循环（若进入）：每轮输入 = 差异报告 + 新增失败（不是全量日志）+ 历史修复
     报告 + baseline 豁免清单；最小变更，新增失败清零即停——预存失败按定义出局。
  5. 最终报告按 SKILL.md「验证与报告」的固定分栏输出：pre-existing 出自 baseline
     （未触碰、不归因于本次迁移）；迁移引入的变化按触点逐项列入「已完成」；残留
     宿主 patch 连同上游 issue/PR 链接列入「待确认/残留风险」（来源格式同
     [README.md](README.md) 卡片规范）。
- **验证**: baseline 采集于任何迁移写入之前（以记录的 `HEAD`/lockfile 哈希等环境
  证据为准）；终局 diff 不含对 baseline 失败文件的顺手修复；报告能对每条失败回答
  「迁移前是否已存在」。
- **来源**: dsh-migrate-bot 无人值守管线的 pre-migration baseline 阶段（机械 pin →
  A/B 审查 → 修复循环 → 补丁报告之上的本地扩展，专管失败归因；撰写时该阶段尚未
  公开推送，公开后请复核此说明——见文末「待确认」）。[#5120](https://github.com/deepseek-ai/deepseek-harness/discussions/5120)
  未覆盖此做法；互补关系：#5120 证明「静态门禁全绿 ≠ 运行时绿」，本条解决另一半
  ——「静态红 ≠ 迁移的错」。

### R-07 · 启动服务竞态：有界重试，不延迟、不加 inject wait

- **类型**: process
- **症状**: 插件启动即轮询依赖服务，与宿主服务就绪窗口竞态；冷启动出现
  `service-unavailable` 循环。分层验证清单第 4 层要求观察此症状，但未给处置配方——
  本条补齐。
- **配方**: 仅限**启动期轮询预期终将就绪的依赖服务**、且被轮询操作只读幂等的场景：
  对 `code: 'service-unavailable'` 做**有界重试**——约 5 次、2 秒退避，总次数与
  总时长有上限，重试参数可注入覆盖（便于测试）；耗尽后明确失败并上报，不无限等待。
  重试前提同 SKILL.md 安全边界：错误可重试、操作幂等、策略允许。若服务在该 cohort
  上根本不存在（永久缺失而非未就绪），走 R-02 的运行时探测回落，不是重试。
- **被否决的替代**: 盲目延迟首次轮询（掩盖竞态而非解决）；把服务加回 inject wait
  （旧 cohort 上入口永久 `pending`，见 R-02）。
- **验证**: 冷启动日志无 `service-unavailable` 循环；注入的重试策略在测试中生效。
- **来源**: [#5120](https://github.com/deepseek-ai/deepseek-harness/discussions/5120)
  第 6 条（dsh-web 迁移记录，boot race 处置；帖内提及决策笔记
  `2026-08-28-task-board-roster-poll-boot-race.md`，未在仓库中定位到公开副本，故不附直链）。
  配方出自原帖作者 zhu1090093659，此处仅按 rollup 格式收录并致谢。

## 分层验证清单

按顺序跑（第 0 层仅采集基线，不设通过门槛）；此后前一层不过不进下一层：

0. **baseline（迁移动手前）**: 在仓库自身依赖状态跑机械套件，记录失败指纹与豁免
   清单（见 R-06）。第 2、3 层（静态与卡片级单测）的失败判定以「相对 baseline
   新增」为准；其余层没有对应基线，保持各自的绝对门槛。
1. **依赖解析**: `pnpm list --depth 0 | grep @deepseek-ai` 版本一致；lockfile 无混合 cohort。
2. **静态**: typecheck + build。注意静态全绿证明不了 wire 契约正确——描述符层的参数漂移在这一层是静默的（[DSH-0.1.2-A1-01](v0.1.2-alpha.1.md)）。
3. **卡片级单测**: 每个命中触点至少一条断言。Remote 调用点覆盖 `ok: false` 的已知业务码、未知码兜底，以及 gateway 层 catch 分支；测试替身编码同一套描述符表，多/缺 key 就 fail，让漂移变成测试失败事件。
4. **真实冷启动**: 完整一轮对话（发消息 → 工具调用 → 回复）。观察日志无 `missed the module table`、无 `service-unavailable` 循环、无入口 `pending`。Web Client 插件另按 [DSH-0.1.2-A1-19](v0.1.2-alpha.1.md) 验证宿主公告资源、bundle 注册、真实挂载与 page error。
5. **跨 cohort**（若做了 R-02）: 旧宿主与新宿主各跑一次第 4 步。
6. **headless**（若命中 #7）: 比对退出码及 stdout/stderr 内容分类（[DSH-0.1.2-A1-05](v0.1.2-alpha.1.md)）。

## 回退

1. 升级前记录 branch/HEAD、resolved 版本、lockfile 与将改配置的 hash；有陌生修改就停止；
2. 在独立 branch/worktree 迁移，不自动 stash/reset/clean/checkout 用户文件；
3. tarball overrides 回退只恢复本次明确拥有的配置与 lockfile 路径，并在执行前展示 diff、取得确认；
4. 第三方 lifecycle script 的任意副作用不能承诺由 Git 回滚；如实列出残留风险；
5. 若问题源于宿主升级，优先切回记录的宿主版本，而不是盲目扩大插件双版本分支。

## 待确认

- 0.1.2 正式版的 dist-tag、final tag 名与 alpha.2 的差异，需在发版后复核本文件全部条目；
- R-01/R-02 的 pnpm 版本敏感性与 R-07 的重试参数（约 5 次 / 2 秒退避）均来自单一实战报告，未在其他仓库复现验证；
- R-06 的 baseline 阶段实现尚未公开推送，公开后复核其来源说明。R-06 属待验证实践（单一管线来源、无多仓库复现）：模式 C 第 0 步是强烈建议的默认动作，可按目标仓库实际情况裁量。
