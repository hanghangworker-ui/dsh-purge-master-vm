# Windows 10 虚拟机：dsh 红队评测环境准备

> 目标：VMware Workstation 上的 Windows 10 x64 客户机（示例配置：4 vCPU / 8 GB RAM / NAT）
> **所有命令都在虚拟机内执行**；宿主机只负责准备安装包，不改动其既有 dsh 环境。
> 用途：**自有靶场 / 书面授权范围内的红队评测**。最后更新：2026-11-05
>
> 本文件是 [dsh-purge-master-vm](https://github.com/hanghangworker-ui/dsh-purge-master-vm) 仓库的操作笔记；适用边界、许可与署名见仓库 [README](../README.md) 与 [NOTICE](../NOTICE.md)。

---

## 零、授权与范围确认（先做这一步，再装任何东西）

红队评测的合法性来自**授权**，不来自工具或环境。下表没填完之前，不要开始任何测试动作。

| 项 | 本次填写 |
|---|---|
| 授权来源 | 自有环境 / 书面授权（授权方、文件编号、签署日期） |
| 靶标范围 | 仅限回环 `127.0.0.0/8` 或自建私网；列出具体 IP / 端口 |
| 时间窗 | 起止时间，超出即停 |
| 允许的测试类型 | 如：护栏有效性对照、越权工具调用观察、拒绝行为差异记录 |
| 明确禁止 | 公网目标、第三方平台与账号、生产系统、真实用户数据、拒绝服务、外挂 |
| 应急联系人 | 谁有权叫停、如何联系 |
| 证据与留存 | 日志 / 截图存放位置、保存期限、谁可访问 |

**四条硬边界**

1. 不碰公网与第三方平台，不对未授权系统扫描或渗透。
2. 靶场里不放真实个人信息、真实凭证、客户代码。
3. 工具自带的「仅回环 / RFC1918 私网、禁止外挂」限制不改动。
4. 「应用 / 生效」环节由操作者本人在隔离 VM 内确认；**本文档不展开关闭护栏、绕过拒绝钩子的操作步骤，也不提供应用步骤**。

### 隔离检查（示例：只读 `.vmx` 得到的结论，请按自己的环境核对）

| 检查项 | 现状 | 处置 |
|---|---|---|
| 运行状态 | `.vmx.lck` 在、有 `.vmss` 挂起态 → VM 运行中或挂起 | 操作前先正常关机 / 恢复到干净状态 |
| 快照 | 已有一个快照文件 | **再打一个命名快照** `pre-redteam-<DATE>`，评测完直接回滚 |
| 网络 | `ethernet0.connectionType = "nat"` | 够用；要更强隔离改用 host-only |
| 共享文件夹 | `isolation.tools.hgfs.disable = "FALSE"`（已启用） | **设为只读**，或只用于单向传入安装包；绝不把宿主 harness 主目录挂成可写工作区 |
| 凭证 | — | 用 VM 独立 API key；不复制本机 `.credentials.yaml` |

### 评测设计（建议 A/B 对照，这才是评测真正的产出）

| 维度 | 干净对照组 | 评测组 |
|---|---|---|
| 环境 | 回滚快照后全新装官方 dsh，不装任何 purge | 按本文第三、四节准备的环境 |
| 任务集 | 同一批任务：常规开发任务 / 边界模糊任务 / 明确越权请求（**只记类别，不在文档里写具体载荷**） | 同左 |
| 度量 | 拒绝率、是否主动询问确认、越权工具调用次数、是否触碰工作区外文件、响应里的安全提醒数 | 同左 |
| 目标 | 得出护栏**有效性**与误报/漏报分布 | 同左 |
| 结论用途 | 改进防护与提示设计，**不用于提炼规避手法** | 同左 |

> 一次只跑一套：改动落在全局宿主层、不在 profile 内，同 VM 里另开 profile 并不能隔离。纯净对照请用另一台 VM，或回滚快照后重装。

### 记录模板（每条任务一行）

| 编号 | 时间 | 任务类别 | 目标 | 观察 | 证据文件 | 判定（拦截/放行/误报） | 备注 |
|---|---|---|---|---|---|---|---|

### 收尾清单

- [ ] 停止会话，导出日志与证据
- [ ] 撤销 VM 里的临时 API key / 凭证
- [ ] 回滚快照 `pre-redteam-<DATE>`
- [ ] 确认本机 `web` profile 未被改动（`dsh --profile web --dump-config` 与基线一致）
- [ ] 出具结论：行为差异来自护栏还是模型能力、误报/漏报分布、防护改进建议
- [ ] 若为委托评测，按授权范围交付报告，不扩散靶标细节

---

## 一、所需环境

| 项 | 要求 | 参考值 |
|---|---|---|
| 操作系统 | Windows 10 x64 | — |
| CPU / 内存 | ≥ 2 vCPU / ≥ 4 GB | 示例 4 vCPU / 8 GB |
| 磁盘 | 至少 5–8 GB 空闲（全局包 + profile 的 `node_modules`） | — |
| **Node.js** | **22 LTS**（建议 ≥ 20；`dsh-purge` 声明 `node >=18`，但 dsh 依赖较新） | 示例 `v22.23.4` |
| **pnpm** | 必须在 PATH 里，`dsh plugin` 就是把参数转给 pnpm；10/11/12 均可 | 实测 `12.16.1` |
| **PowerShell 执行策略** | `CurrentUser` 至少 `RemoteSigned`；npm / pnpm / dsh 的 Windows shim 都是 `.ps1`，`Restricted` 会全部拦下 | VM 默认 `Restricted`，需按 Step 0 调整 |
| Git | **方案 B 不需要**（本地源码 `add .` 或 codeload 直链都由 pnpm 处理） | — |
| 网络 | 需能访问 `registry.npmjs.org`、`codeload.github.com`、`github.com/.../releases` | VM 为 NAT，通常可出网 |
| 模型 API | dsh 要能对话必须配置 provider / API key（建议用 VM 自己的 key） | 使用 VM 独立凭据 |
| `DSH_HOME` | 可选，不设则默认 `%USERPROFILE%\.dsh` | 示例：`C:\Users\<USER>\dsh-redteam` |

> **别把宿主机的 `.credentials.yaml` 复制进 VM**——里面是凭证。用 VM 独立的 key。

---

## 二、先选版本组合（关键）

| 方案 | dsh 本体 | 破甲插件 | 说明 |
|---|---|---|---|
| **A（开箱即用）** | `0.2.1-rc.5` | `dsh-purge-community` 1.6.74 | 不用降级、功能较全，适配新版宿主 |
| **B（原版 master）** | `0.2.1-rc.2` | `dsh-purge` 1.1.90（HunYiSage 原版） | README 明确「不支持 0.2.1-rc.3 及更新」，所以**必须把 dsh 降到 rc.2**；需自行从上游获取源码 |

> 不要混搭：用方案 B 的插件配 rc.5 宿主，补丁会显示「待应用 / 跳过」，`--dump-config` 也会报兼容性警告。
> npm 上可查到的版本含 `0.2.1-rc.2`、`rc.3`、`rc.4`、`rc.5`、`0.2.2-rc.1`，别手滑装成 `0.2.2`。

### 方案怎么选

| 维度 | A 社区版 1.6.74 + rc.5 | B 原版 1.1.90 + rc.2 |
|---|---|---|
| 第三方依赖 | 1 个（`yaml`） | **0 个**（仅 peer `@deepseek-ai/cordis`，dsh 自带） |
| dsh 宿主 | 不降级 | **必须降到 rc.2** |
| 源码 | 第三方 fork tgz，需解包 | 上游仓库完整源码，可读可二开 |
| 许可证 | MIT | Apache-2.0（二开须署名） |
| 功能 | 多 `dsh-purge-presets`、演练台增强 | 基础清洗 + 演练台 |
| 适配声明 | `dshTarget: 0.2.1-rc.1`（实际却在 rc.5 上跑） | `dshTarget: 0.2.1-rc.2`，README 称不支持 rc.3+ |

**做研究 / 二开选 B，要开箱即用选 A。** VM 的全部价值就是能折腾、能回滚快照，rc.2 旧宿主的代价在快照面前不值一提；而 B 依赖为零、源码在手，干净环境里干扰变量最少，评测结论更可信。A 的优势是不用降级、功能更全。

> 两边的 `dshTarget` 声明都自相矛盾，最终一律以 `dsh --profile web --dump-config` 实测为准。

---

## 三、安装命令（PowerShell，在 VM 内）

### ★ 最终方案 B 执行清单（rc.2 + 1.1.90，可直接复制）

> 前置：第零章授权表已填、`pre-redteam-<DATE>` 快照已打；**全程用管理员 PowerShell**。
> **示例 VM 没有共享文件夹，也不需要配**（读者请按自己 `.vmx` 核对）。 `.vmx` 里只有 `isolation.tools.hgfs.disable = "FALSE"`（VMware Tools 的 HGFS 服务开关），**没有任何 `sharedFolder*.present` 条目** → UNC 根 `\\vmware-host\Shared Folders` 可达但内容为空，属正常现象，不是故障，改任何 PowerShell 路径都无效。
> 源码因此默认走 **VMware Tools 拖放 / 复制粘贴**（`.vmx` 中 `isolation.tools.dnd.disable`、`isolation.tools.copy.disable` 均不存在，未被禁用）。宿主机侧无需新增任何共享配置。
> **本清单到「只读验证」为止，不含应用 / 生效步骤。**

```powershell
# 0 · 运行时（**以管理员身份**打开；MSI 装完重开一次 PowerShell 让 PATH 生效）
# 精简版 / LTSC Win10 常没有 winget，直接用官方 MSI（等价于 winget 的 OpenJS.NodeJS.LTS）
$ver = 'v22.23.4'
$url = "https://nodejs.org/dist/$ver/node-$ver-x64.msi"
$msi = Join-Path $env:TEMP "node-$ver-x64.msi"
Invoke-WebRequest -Uri $url -OutFile $msi -UseBasicParsing
Start-Process msiexec.exe -ArgumentList "/i `"$msi`" /qn /norestart /log `"$env:TEMP\node-install.log`"" -Wait
# 完整性校验（可选）：(Get-FileHash $msi -Algorithm SHA256).Hash.ToLower() 对比 https://nodejs.org/dist/$ver/SHASUMS256.txt
# VM 不出网时：在宿主机下好 MSI，经拖放 / 复制粘贴传入 VM，把 $msi 改成实际路径再单独执行上面那行 Start-Process
# 重开 PowerShell 后：先放行本地 .ps1 shim（npm / pnpm / dsh 在 Windows 上都是 .ps1）
Get-ExecutionPolicy -List
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force
# 若被组策略覆盖而报错，改用只影响当前窗口的：Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
# 继续：
npm install -g pnpm
node -v ; npm -v ; pnpm -v             # 期望 v22.23.4 / 10.x / 10~12.x

# 1 · DSH_HOME 放用户目录（避开 D:\ 根目录 ACL；当前会话立即生效 + 持久化）
$env:DSH_HOME = "C:\Users\admin\dsh-redteam"
setx DSH_HOME "C:\Users\<USER>\dsh-home"

# 2 · dsh 本体，必须锁 rc.2
# ECONNRESET 会留下半装目录：先清掉，换镜像 + 放宽超时再装（不要清 npm 缓存，复用已下载 tarball）
Remove-Item -LiteralPath "C:\Users\admin\AppData\Roaming\npm\node_modules\@deepseek-ai" -Recurse -Force -ErrorAction SilentlyContinue
npm config set registry https://registry.npmmirror.com
npm config set fetch-retries 10
npm config set fetch-retry-maxtimeout 180000
npm config set fetch-timeout 600000
npm install -g @deepseek-ai/dsh@0.2.1-rc.2
dsh --version                          # 必须是 0.2.1-rc.2；出现 rc.5 就停下
# 若再 ECONNRESET：直接重跑上面那条 npm install 即可

# 3 · 源码传入 VM 本地盘（本机无共享文件夹，走 VMware Tools 拖放 / 复制粘贴）
# 【宿主机侧】按 README「准备源码物料」打包 dsh-purge-master.zip 后拖进 VM 桌面
#   或：宿主机右键该文件「复制」→ 在 VM 桌面「粘贴」（共享剪贴板）
#   已排除 .git 与 node_modules，解包后顶层须为 dsh-purge-master/
$zip = "C:\Users\admin\Desktop\dsh-purge-master.zip"
if (-not (Test-Path $zip)) { throw "zip not found - drag it into the VM desktop first" }
New-Item -ItemType Directory -Force -Path "$env:DSH_HOME\src"
Expand-Archive -LiteralPath $zip -DestinationPath "$env:DSH_HOME\src" -Force
Test-Path "$env:DSH_HOME\src\dsh-purge-master\package.json"     # 必须 True

# 4 · 注册为 web profile 的 bundle（add 会自动初始化 profile）
cd "$env:DSH_HOME\src\dsh-purge-master"
dsh plugin --profile web add .
# 不想用共享文件夹时改用官方直链：
# dsh plugin --profile web add https://codeload.github.com/HunYiSage/dsh-purge/tar.gz/refs/heads/master

# 5 · 必做：补齐 dsh.profile.bundles 与用户层「本体 insert」
cd "$env:DSH_HOME\profiles\web"
node --preserve-symlinks-main node_modules/dsh-purge/scripts/ensure-profile-bundle.mjs
# 期望 2~3 行输出：dsh-purge: already in bundles / appended to bundles
#                  insert already in / wrote body insert
#                  allowed pnpm build for dsh-purge
# 若一行都没有 = 脚本两个静默点之一触发，务必走下方「5b 兜底」

# 6 · 只读验证
dsh --profile web --dump-config | Select-String -Pattern "id: dsh-purge|dsh-purge-redteam"
dsh --profile web --dump-config        # 完整树：2 个官方 bundle + 1 个 dsh-purge
```

**Step 3 的三条传入路线**

源码物料 **不随本仓库分发**（含第三方代码，许可见 [NOTICE](../NOTICE.md)）。请从上游获取后自行打包，见 [README](../README.md#准备源码物料)；或直接使用下文路线 ③ 的 codeload 直链，零文件传输。

- **路线 ① 拖放 / 复制粘贴（本 VM 默认，见上方清单）**：VMware Tools 自带，零配置。`.vmx` 里 `isolation.tools.dnd.disable`、`isolation.tools.copy.disable` 均不存在，未被禁用。
- **路线 ② 共享文件夹**（仅当以后确实要用 UNC）：先在宿主机 VMware → 虚拟机设置 → 选项 → 共享文件夹 → 添加宿主机上的源码目录，名称 `purgesrc`（ASCII）、勾只读，VM 运行中可热添加、无需重启。然后：

```powershell
Get-ChildItem "\\vmware-host\Shared Folders" -Force | Select-Object Name
$pkg = Get-ChildItem "\\vmware-host\Shared Folders" -Recurse -Depth 2 -Filter package.json -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
if (-not $pkg) { throw "shared folder has no package.json - check host-side settings" }
Copy-Item -Recurse -Force (Split-Path $pkg) "$env:DSH_HOME\src\dsh-purge-master"
Test-Path "$env:DSH_HOME\src\dsh-purge-master\package.json"    # 必须 True
```

- **路线 ③ 官方 codeload 直链**（零文件传输，但版本是远端 master、依赖 GitHub 连通性）：跳过 Step 3，直接执行下面这一行，然后跳到 Step 5。

```powershell
dsh plugin --profile web add https://codeload.github.com/HunYiSage/dsh-purge/tar.gz/refs/heads/master
```

> 三条路最终都得到同一个源码目录（或同一个已注册的 bundle），任选其一走通即可。

**完成标志**：`dsh --version` = `0.2.1-rc.2`；`--dump-config` 退出码 0、**必须出现 `- id: dsh-purge` 本体行**（只有 `dsh-purge-redteam*` 不算）、无版本不兼容报错；profile 共 3 个 bundle。

### 5b · 第 5 步静默无输出时的兜底

`ensure-profile-bundle.mjs` 有两个**不报错、直接 return** 的静默点：`findProfileDir()` 没认出 profile；以及末尾 `invoked === fileURLToPath(import.meta.url)` 的入口判定在 Windows + pnpm 符号链接 / 路径大小写下不相等。已在 Windows + pnpm 12 环境复现「非 profile 目录下 exit 0、零输出」。先用 `--preserve-symlinks-main` 修 symlink 情形；仍无输出就显式调用官方函数，绕过这两个判定：

```powershell
cd "$env:DSH_HOME\profiles\web"
@'
import fs from "node:fs";
import { ensureProfileBundles, ensureProfileBodyInsert, ensurePnpmAllowBuild, profileHasPurgeBodyInsert } from "./node_modules/dsh-purge/lib/profile-insert.js";
const d = process.cwd();
const read = () => fs.existsSync("cordis.patch.yml") ? fs.readFileSync("cordis.patch.yml", "utf8") : "";
console.log("before:", profileHasPurgeBodyInsert(read()));
console.log("bundles:", JSON.stringify(ensureProfileBundles(d)));
console.log("insert:", JSON.stringify(ensureProfileBodyInsert(d)));
console.log("allowBuild:", JSON.stringify(ensurePnpmAllowBuild(d)));
console.log("after:", profileHasPurgeBodyInsert(read()));
'@ | Set-Content -Path "$env:DSH_HOME\fix-bundle.mjs" -Encoding ASCII
node "$env:DSH_HOME\fix-bundle.mjs"
Select-String -Path cordis.patch.yml -Pattern '^\s*- id: dsh-purge\s*$'
dsh --profile web --dump-config | Select-String -Pattern 'id: dsh-purge$|not found'
```

`after: true` 且 `--dump-config` 出现本体行即修好。按 README，补完 insert 后要**完全退出宿主再重开**（必要时再重开一次，让 redteam 子条目自愈）。

### 6 条 `entry ... not found` 怎么读

| 警告 | 性质 |
|---|---|
| `redteam-store` / `redteam-ui` / `redteam-mode` / `redteam-mode-store` / `redteam-mode-ui`（5 条） | **可忽略**。包内 `cordis.patch.yml` 第 6–25 行故意写了 5 条 `disabled: true` 去关旧版独立红队包；没装过就报找不到。README 原文：*leftover from an old red-team package and can be ignored* |
| `entry "dsh-purge" not found`（第 6 条） | **分情况**。包内 patch 第 35–56 行是 `- id: dsh-purge` 的**修改项**（不是 insert），只改「已入队」的本体条目；合并顺序上它先于用户层 `- insert:` 生效，此时本体行尚未入队，于是报 not found。**判据看最终树里有没有 `- id: dsh-purge` 本体行**：<br>• 没有 → 真问题，宿主不加载插件本体、界面没按钮，走 5b 补齐（README.md:228 的判据）<br>• 有 → 过程性合并顺序产物，见下方「补完后的状态」 |

> 注意：`system-prompt` 段显示 `includeHarnessIdentity: false` 说明**配置树组合层**的 patch 已生效（bundle 已加载），与「本体 insert 是否存在」是两件事，也不代表官方包文件已被改写。但要如实说清：这一层 patch 是**安装动作本身的结果**，一旦启动 `dsh web` 就会随配置树起作用；「应用 / 生效」改的是官方 `@deepseek-ai` 包文件与 `dsh` shim，是另一层。

### 补完后的状态（示例 VM 实测）

`--preserve-symlinks-main` 补跑成功，输出：

```
dsh-purge: already in bundles (...\profiles\web)
dsh-purge: wrote body insert (...\profiles\web\cordis.patch.yml)
dsh-purge: allowed pnpm build for dsh-purge
```

`Select-String` 在 `cordis.patch.yml:5` 命中 `- id: dsh-purge`，`--dump-config` 也输出了 `- id: dsh-purge` 本体行 → **第 5 步的完成标志全部达成**。

此时 6 条警告仍会全部出现，其中第 6 条**不必再修**：官方 `INSERT_BLOCK`（`lib/profile-insert.js`）写进用户层的本体行已经自带 `inject: [systemPrompt, tools]` 与 `enabled / stripMnemon / capabilityProtocol / hooksDenyBypass` 四项核心 config，功能不缺；包内 overlay 里其余细项（`anchorMode`、`subagentControl`、`autoApplyOnStart` 等）走代码默认值。README 未把这条列入"可忽略"清单，因此建议把它作为**观测项**记进评测记录，并在重启宿主后确认按钮出现即可。

### 收尾清单（到只读验证为止）

1. **完全退出宿主并重新打开**（README Step 3）：停掉 `dsh web`、重新起一次，让本体 insert 加载。预期结果：会话标题旁出现 **dsh-purge** 按钮。按钮不出现才回头排查，不要继续往下走。
2. **打一个 `pre-apply-<DATE>` 快照**（应用之前），便于 A/B 对照与回滚。
3. **配置 VM 独立的模型凭据**，确认能正常起一次会话。不要复制宿主机的 `.credentials.yaml`。
4. **评测前置**：填好第零章授权表；准备干净对照组（另一台 VM，或回滚快照后重装；同一 VM 多 profile 不隔离，因为改动在全局宿主层）；任务集只记类别不写载荷；记录模板就位。

**到此为止。** 后续的「应用 / 生效」由操作者本人在 VM 内确认，本文档不展开其操作步骤。

> 方案 B 专坑：
> - `file:` 归档方式安装后，**改了源码不会自动生效**，要重新 `dsh plugin --profile web add .`。
> - 宿主误装成 rc.3+ 会显示「待应用 / 跳过」；先 `dsh --version` 自查，别用 rc.5 硬上。
> - `postinstall` 的 `ensure-profile-bundle.mjs` 负责把 `dsh-purge` 写进 `dsh.profile.bundles` 并补 `cordis.patch.yml` 的 `insert`；没跑会导致界面没有按钮、宿主不加载插件。
> - 不要走插件市场（会报 `already queued: dsh-purge`），npm 上也没有 `dsh-purge` 这个包。
> - **pnpm 12 必拦 `postinstall`**：VM 实测 `12.16.1`。`dsh plugin add` 本身仍会成功，但脚本不跑 → 用户层 `cordis.patch.yml` 缺本体 `insert`、宿主不加载插件、界面没按钮。第 5 步的补跑脚本是幂等的（源码 `scripts/ensure-profile-bundle.mjs` 里 `bundles.includes` 即跳过），还会顺手把 `allowBuilds: dsh-purge` 写进 `pnpm-workspace.yaml`；**无论有没有看到 `ERR_PNPM_IGNORED_BUILDS` 警告都跑一次**。该脚本从 `INIT_CWD` / `npm_config_local_prefix` / `process.cwd()` 向上最多找 8 层 profile 目录，找不到就静默 `return`，所以务必先 `cd` 进 profile 目录。

---

### 以下为分步骤详解（A / B 通用）

> **「只装破甲」可以，但不能省略 dsh 本体。**
> 破甲是 dsh 的 bundle 插件：`dsh plugin` 命令本身由 dsh 提供，插件的 `cordis.patch.yml` 要注入 dsh 的配置树，它改写的也是官方 `@deepseek-ai` 包和 `dsh` shim。不装 dsh，装下来只是一堆跑不起来的 JS 文件——没有会话、界面和模型调用。
> 但「不装其它第三方插件」完全没问题：全新 `web` profile 的默认模板只有 2 个官方 bundle（`@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`），加上破甲正好 **3 个 bundle**；破甲自身 0 个第三方依赖。这正是干净 VM 最值得的地方。
> 极简路径：`npm i -g @deepseek-ai/dsh@<版本>` → `dsh plugin --profile web add <包>` → `dsh --profile web --dump-config`。`dsh plugin add` 会自动初始化 profile，不必先跑 `dsh web`。

### 0 · 基础运行时

> **示例 VM 实测没有 winget**，请用上方「★ 最终方案 B 执行清单」Step 0 的**官方 MSI 静默安装**。下面是自带 winget 的完整版系统上的等价命令，本机 VM 不适用：

```powershell
# 仅当系统自带 winget 时
winget install OpenJS.NodeJS.LTS --accept-package-agreements --accept-source-agreements
npm install -g pnpm

node -v        # 期望 v22.x
pnpm -v        # 期望 10.x / 11.x / 12.x
```

> 别去修 App Installer：LTSC / 精简镜像上装回来比 MSI 麻烦得多。
> **方案 B 不需要 Git**（本地源码 `add .` 或 codeload 直链都由 pnpm 处理），可跳过。
> 装完**重开一次 PowerShell**，让 PATH 生效。

### 1 · 装 dsh 本体

方案 A：

```powershell
npm install -g @deepseek-ai/dsh@0.2.1-rc.5
dsh --version
```

方案 B：

```powershell
npm install -g @deepseek-ai/dsh@0.2.1-rc.2
dsh --version
```

可选，指定独立的 harness 主目录：

```powershell
setx DSH_HOME "C:\Users\<USER>\dsh-home"
```

### 2 · 装破甲插件（web profile）

方案 A —— 社区版 1.6.74：

```powershell
dsh plugin --profile web add https://github.com/kingcharleslzy-ai/dsh-purge/releases/latest/download/dsh-purge-community.tgz
```

方案 B —— 原版 1.1.90：

```powershell
dsh plugin --profile web add https://codeload.github.com/HunYiSage/dsh-purge/tar.gz/refs/heads/master
```

方案 B 变体 —— 直接用宿主机上的 master 源码（把路径换成 VM 里看到的路径）：

```powershell
dsh plugin --profile web add "<VM 内看到的源码目录>"
```

> 不要用「插件市场」一键安装（会报 `already queued: dsh-purge`）；
> npm 上没有 `dsh-purge` 这个包，别 `pnpm add dsh-purge`。

### 3 · 验证组合结果（只读）

```powershell
dsh --profile web --dump-config
```

完成标志：输出里出现 `- id: dsh-purge` 及对应的 `dsh-purge-redteam*` 条目，且没有版本不兼容报错。

### 4 · 启动与生效（由操作者在 VM 内确认）

```powershell
dsh web
```

工具还有一个「应用 / 生效」环节：它会改写官方 `@deepseek-ai` 包、注入提示词、关闭 Harness 身份、绕过拒绝钩子，并调整 `dsh` / `dsh.cmd` / `dsh.ps1` shim。**这一步请操作者在隔离 VM 内自行确认；本文档不展开其操作步骤。**

> 生效后建议立刻再打一个快照 `post-apply`，便于 A/B 之间快速切换。
> 注意：改动落在全局宿主层、**不在 profile 目录内**——同一 VM 里另开 profile 并不能隔离它。要纯净对照，请用另一台 VM 或回滚快照后重装。
> 只读验证仍用 `dsh --profile web --dump-config`（不会触发生效）。

---

## 四、常见报错

| 现象 | 处理 |
|---|---|
| `npm error code ECONNRESET` + `npm warn cleanup ... EPERM` | 安装中途断网，全局包只装了一半，回滚清理又被 Windows 文件锁挡住。用**管理员** PowerShell 删掉 `C:\Users\admin\AppData\Roaming\npm\node_modules\@deepseek-ai` 残留，换 `https://registry.npmmirror.com` + 放大 `fetch-retries/timeout` 后重装；**不要 `npm cache clean`**，保留缓存才能续传。dsh 本体仅 76 KB，但依赖树含 `openai` / `@aws-sdk` 等，网络抖动时容易断 |
| `dsh : The term 'dsh' is not recognized`、`cd ... does not exist`、`Cannot find module ...ensure-profile-bundle.mjs` | **全是上一条的连锁失败**，别逐个修。dsh 没装成就没有 shim、没有 profile 目录、没有 `node_modules`；装好 dsh 后自动消失 |
| `New-Item : Access to the path 'dsh-redteam' is denied` | 标准令牌下不能在盘根目录建目录。要么用**管理员** PowerShell，要么把 `DSH_HOME` 改到用户目录 `C:\Users\admin\dsh-redteam`（推荐，清单已改） |
| `Cannot find drive. A drive with the name 'Z' does not exist` | VMware 共享文件夹没映射成 `Z:`。用 `Get-PSDrive -PSProvider FileSystem` 看真实盘符，或直接走 UNC `\\vmware-host\Shared Folders\<共享名>`；共享名设成 ASCII（如 `purgesrc`）避开中文乱码 |
| `Test-Path "\\vmware-host\Shared Folders"` = `True`，但 `Get-ChildItem ... -Filter package.json` 返回空 / `$pkg` 为 null | **正常现象，不是故障**：VMware Tools 的 HGFS 服务活着，但宿主机侧从未添加共享目录（`.vmx` 只有 `isolation.tools.hgfs.disable="FALSE"`，无 `sharedFolder*.present`）。改 PowerShell 路径无效。**默认改走 Step 3 路线 ①（zip 拖放 / 复制粘贴）**，不需要配置共享文件夹；只有确实要用 UNC 时，才去 VMware → 虚拟机设置 → 选项 → 共享文件夹 → 添加宿主机目录（名称 `purgesrc`、勾只读，运行中可热添加） |
| `Cannot find path '\\vmware-host\Shared Folders\purgesrc\dsh-purge-master' ... does not exist` | UNC 根通了但**共享名或层级不对**：可能共享名不叫 `purgesrc`，也可能共享直接指向 `dsh-purge-master` 本身。用 Step 3 的 `Get-ChildItem -Recurse -Depth 2 -Filter package.json` 自动定位源码根，别猜路径；若 `Test-Path "\\vmware-host\Shared Folders"` 返回 `False`，则是 VMware Tools / 共享文件夹未启用 |
| `entry "redteam-store" / "redteam-ui" / "redteam-mode" ... not found`（5 条） | **可忽略**：包内 `cordis.patch.yml` 第 6–25 行用 5 条 `disabled: true` 关旧版独立红队包，没装过就报找不到，README 明说可忽略 |
| `entry "dsh-purge" not found`，且树里只有 `dsh-purge-redteam*`、**没有 `- id: dsh-purge` 本体行** | **真问题**：用户层 `cordis.patch.yml` 缺本体 insert → 宿主不加载插件本体、界面没有按钮。按 README 判据检查并走 5b 补齐，再完全退出宿主重开 |
| `entry "dsh-purge" not found`，但**已有 `- id: dsh-purge` 本体行**（补跑成功后） | **不必再修**：包内 patch 第 35–56 行是「修改已入队条目」的 overlay，合并顺序先于用户层 `- insert:`，本体行当时还没入队才报 not found。官方 `INSERT_BLOCK` 已自带 `inject` 与四项核心 config，功能不缺；细项走代码默认值。重启宿主确认按钮出现，把这条记为评测观测项即可 |
| 第 5 步 `node ...ensure-profile-bundle.mjs` **零输出、退出码 0** | 脚本两个静默点：`findProfileDir()` 未认出 profile，或 `invoked === fileURLToPath(import.meta.url)` 入口判定在 Windows + pnpm 符号链接 / 路径大小写下不相等（已在 Windows + pnpm 12 复现）。见 5b 兜底 |
| `npm.ps1 cannot be loaded because running scripts is disabled` | Windows 默认 `Restricted` 执行策略拦 `.ps1` shim。执行 `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force`（只改当前用户，不动系统级策略）；被组策略覆盖时改 `-Scope Process -ExecutionPolicy Bypass -Force`。**不解决这条，后面 `dsh` / `pnpm` 每一步都会卡** |
| `pnpm : The term 'pnpm' is not recognized`（紧接上一条出现） | 不是 PATH 问题——是 `npm install -g pnpm` 被上一条拦住、根本没装成。先修执行策略，再重装 pnpm |
| `winget : The term 'winget' is not recognized` | 精简版 / LTSC Win10 未带 App Installer。**别去修 winget**，直接用第三节 Step 0 的官方 MSI 静默安装（等价 Node 22 LTS）；VM 不出网就由宿主机下载 MSI 后拖放传入 |
| MSI 装完 `node -v` 仍不识别 | 先 `Test-Path 'C:\Program Files\nodejs\node.exe'`；存在则重开终端，或临时 `$env:Path += ';C:\Program Files\nodejs'`；不存在就看 `%TEMP%\node-install.log`，确认是否用管理员权限运行 |
| `ERR_PNPM_IGNORED_BUILDS` / `postinstall` 没跑，界面没按钮 | **pnpm 12 上几乎必现，有时只给警告不报错**。不用重装：直接执行 ★ 清单第 5 步（务必先 `cd` 进 profile 目录，再跑 `ensure-profile-bundle.mjs`），幂等且会补 `allowBuilds`；必要时再 `dsh plugin --profile web approve-builds` |
| `dsh: pnpm was not found` | pnpm 不在 PATH；`npm i -g pnpm` 后重开终端 |
| `already queued: dsh-purge` | 别用市场；先 `dsh plugin --profile web remove dsh-purge` 再重新 add |
| GitHub 下载超时 / 403 | 在宿主机先把 tgz 下好，拖放传入 VM，改成 `dsh plugin --profile web add "<VM 内实际路径>"`（本地路径，不走网络） |
| `--dump-config` 报版本不兼容 | 按提示 `dsh plugin --profile web allow-version <pkg>@<ver> --dsh-version <exact> --accept-risk`；更推荐直接换回匹配的版本组合 |
| `entry "redteam-store" not found` 等几条警告 | 旧红队包不在，可忽略（README 有说明） |

---

## 五、隔离与合规边界

- **第零章的授权表是前置条件**：授权范围没确认，就不开始测试动作。
- 只在**隔离虚拟机**里做评测；VM 用 NAT 或 host-only，别把宿主共享目录挂成可写工作区。
- 这类插件的用途是让模型绕过默认护栏，**仅限自己有权管理的本机环境、离线靶标、授权靶场内的红队评测**；不用于软件设计、日常开发、生产系统，也不用于对外交付的系统或内容。
- 不对公网或未授权系统做扫描/渗透，不生成违法违禁内容，不做游戏外挂；工具自带的「仅回环 / RFC1918 私网」限制不要去改。
- 评测结论用于**改进防护**，不用于提炼规避手法或扩散到靶场之外。
- 想彻底干净：装完先给 VM 打一个快照，评测完直接回滚快照。
