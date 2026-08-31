# publish-playbook · 打包、发布与分发配方

> 按需加载的实操配方，承接 [../SKILL.md](../SKILL.md) 的决策流程。
> 所有结论来自 omdsh-dev 组织 17 个插件仓库两轮真实发布实践（rc.2 → alpha.1 → alpha.2），
> 场景未覆盖时以一手来源为准并标注「待确认」。

## 未发布 cohort 安装（R-01 配方）

0.1.2-alpha.* 只在 GitHub 发布，npm 查询返回 404。消费者侧的隔离安装：

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git /tmp/dsh-build
cd /tmp/dsh-build && git checkout dsh-v0.1.2-alpha.2
pnpm install && pnpm run build
mkdir -p ~/.dsh-cohorts/0.1.2-alpha.2
pnpm -r exec pnpm pack --pack-destination ~/.dsh-cohorts/0.1.2-alpha.2
```

manifest 里 range 写 `^0.1.2-alpha.2`，并用 `overrides` 钉到 `file:` tarball；正式发版后删掉 overrides 即回到 registry 解析。

> **待确认（单一实战报告，未复现）**：pnpm 11.9.0 对 file: tarball 的传递依赖在有第三方
> peer 时会绕过 overrides 去 registry 找不存在的版本，报告称钉 `packageManager: pnpm@11.24.0`
> 才解析正确。落地前先在目标仓库做最小复现，验证后回填结论。

## 双兼容写法（alpha 时代核心策略）

公开仓库的 devDependencies 用 npm 发布线（当前 0.1.1-rc.2）作为类型基线；代码同时要在
GitHub tag 的本地 harness 上运行。对签名漂移的 API，按“npm 发布线类型为准、alpha 运行时
语义不变”折中。真实案例：`rpc.handle` 第三参数在 0.1.2-alpha.1 起被移除（认证改由
connection 统一处理），但 rc.2 类型仍要求它：

```ts
// devDependencies 基线是 npm 发布线（rc.2），其 handle() 类型要求第三参数；
// harness 0.1.2-alpha.1 起的 handle() 忽略该参数。保留它使仓库对 rc.2 类型
// 可 typecheck，同时 alpha 运行时行为不变。
const dispose = connection.rpc.handle(
  '/tariff',
  handler,
  { authority: 'loopback' },
)
```

- 只对**签名漂移且运行时忽略多余参数**的 API 用此折中；语义变化的 API 必须按
  [plugin-upgrade](https://github.com/oh-my-dsh/dsh-plugin-upgrade-skill/blob/main/skills/plugin-upgrade/SKILL.md) 的版本卡片迁移，不做静默折中；
- 类型导入（`import type`）编译期擦除，跨 cohort 无运行时负担；
- 不要把本机 junction/file: 绝对路径写进提交的 manifest。

## CI 与发布门禁（未发布 cohort）

- cohort tarball 用 actions cache 物化（以 manifest hash 为 key），所有 pnpm 消费 job 共享，
  避免 frozen lockfile 记录机器相关路径导致干净 runner 缺 store；
- `pnpm/action-setup` 不写死 `version`，让 `packageManager` 成为唯一版本来源；
- 发布 workflow 加 `NPM_PUBLISH_ENABLED` 开关：tag 触发仍跑全部门禁与 smoke，但跳过
  npm publish，直到 cohort 正式发布。

## 发布语义门禁配方（release workflow）

发布前按序跑以下检查，任一失败即停（SKILL 第 4 步的四条不变量）：

```sh
VERSION="$(node -p "require('./package.json').version")"
# 1) GitHub Release tag 必须等于 v${VERSION}
[ "$RELEASE_TAG" = "v$VERSION" ] || { echo "tag mismatch"; exit 1; }
# 2) prerelease 状态一致（'-' 在 '+' build metadata 之前）
V_PRERELEASE="$(node -e 'console.log(process.argv[1].split("+")[0].includes("-") ? "true" : "false")' "$VERSION")"
[ "$RELEASE_PRERELEASE" = "$V_PRERELEASE" ] || { echo "prerelease state mismatch"; exit 1; }
# 3) 分流 dist-tag：prerelease 只进项目声明的非 latest tag（NEXT_TAG 由项目自定），stable 才进 latest
if [ "$RELEASE_PRERELEASE" = "true" ]; then NPM_TAG="$NEXT_TAG"; else NPM_TAG="latest"; fi
# 4) stable 发布前拒绝把 latest 回退到更低版本（semver 比较）
if [ "$NPM_TAG" = "latest" ]; then
  CURRENT="$(npm view "$PKG" dist-tags.latest 2>/dev/null || echo 0.0.0)"
  node -e "const semver=require('semver'); if (semver.lt(process.argv[1], process.argv[2])) { console.error('refusing to move latest backwards'); process.exit(1) }" "$VERSION" "$CURRENT"
fi
npm publish --access public --tag "$NPM_TAG"
```

- 宿主验收矩阵与发布通道一致：prerelease 通道锁 alpha 系 tag、stable 通道锁 rc 系 tag，
  **绝不跟随 master/main 冒名验收**；
- Web Client 插件的 publish 前 smoke 必须覆盖：宿主启动图（`window.__DSH_BOOT__`）公告的
  bundle 入口可访问、bundle 注册成功、DOM 挂载完成、无 page error；`--dump-config` 只证明
  row 存在，不代替本项；
- 可复核实现参考：[dsh-genui#86](https://github.com/omdsh-dev/dsh-genui/pull/86)、
  [dsh-annotation#40](https://github.com/omdsh-dev/dsh-annotation/pull/40)（两者实现了
  第 1–3 条与双宿主 smoke；第 4 条 latest 回退防护为社区补充建议，参考实现中尚无）。

## 真实坑位清单（两轮实践）

| 坑 | 症状 | 处置 |
|---|---|---|
| PATH 上没有 pnpm（只有 corepack） | 构建脚本里嵌套 `pnpm --filter …` 报 `'pnpm' is not recognized` | 生成 `pnpm.cmd` shim 转发 corepack，前置到 PATH；启动/构建包装器自举该 shim |
| Windows PowerShell 5.1 调 `npm` 解析到 `npm.ps1` | 参数错乱（`Unknown command: "pm"`） | 包装脚本显式调用 `npm.cmd` / `pnpm.cmd` |
| PowerShell 5.1 参数默认值里 `$PSScriptRoot` 为空（带 `[CmdletBinding()]` 时） | `Join-Path` 报空字符串 | 默认值移到脚本体内解析 |
| PowerShell 的只读自动变量 `$Host` | 参数名 `-Host` 覆盖失败 | 改名 `-BindHost` 等 |
| `git rebase --continue` 卡编辑器 | 无 TTY 时挂起 | `GIT_EDITOR=true`（或 `core.editor=true`）再继续 |
| 远端前进导致推送被拒 | `[ahead 1, behind 1]` | `git pull --rebase` 后 `--force-with-lease` 重推，绝不裸 `--force` |

## 回滚配方

1. 发布前打 tag 并记录 lockfile/composition 基线 hash；
2. GitHub 直装轨：删除/移动 tag，消费者按需重新指向旧 commit；
3. 隔离工作区（branch/worktree）做迁移与发布，不与功能改动混在一个提交；
4. 失败时只回滚本次拥有的路径（tag、lockfile、manifest），报告第三方安装脚本的残留副作用。

## 待确认

- 0.1.2 正式版 dist-tag 与 final tag 名发布后，复核“未发布 cohort”各条配方；
- pnpm 版本敏感性（见上）来自单一实战报告，复现前标待确认。
