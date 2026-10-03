# BeamMP-Offline — Master Context / 项目记忆

> **这份文档是这个项目的"大脑"。** 任何新的 AI 会话（或新维护者）在继续本项目的开发前，
> 必须先完整阅读本文件。读完即可恢复全部上下文，正确地进行开发与上游合并。

---

## 1. 项目初衷（为什么存在这个项目）

- **降低 BeamMP 的使用门槛**：原版 BeamMP 要求玩家在 BeamMP 论坛注册账号、获取
  认证 key，服务端也要到 keymaster 申请 AuthKey，所有认证都依赖 BeamMP 的在线后端
  （auth.beammp.com / backend.beammp.com）。
- **目标**：改造成**纯离线版本** —— 没有任何互联网的玩家（内网/局域网/无外网环境）
  也能联机：装好 mod → 启动器 → 输入服务器 IP → 直接玩。
- **发布方式**：GitHub 三个仓库 + GitHub Actions 自动构建（本地无编译环境）。
- **维护方式**：持续跟踪上游（BeamMP 官方仓库）的更新，通过 git merge 同步上游代码，
  同时**永久保留本项目的离线化补丁**。

## 2. 仓库结构（三个仓库，dksslq 账号下）

| 仓库 | 上游来源 | 内容 |
|---|---|---|
| `dksslq/BeamMP-Offline` | BeamMP/BeamMP（meta 仓库） | 客户端 Lua/Vue mod 源码 + **本记忆文档** + mod 打包 CI |
| `dksslq/BeamMP-Server-Offline` | BeamMP/BeamMP-Server | C++ 服务端（离线版） |
| `dksslq/BeamMP-Launcher-Offline` | BeamMP/BeamMP-Launcher | C++ 启动器（离线版） |

- 每个仓库都保留完整上游 git 历史，并添加 `upstream` remote 指向官方仓库。
- 上游分支：官方 `master`（BeamMP-Server/BeamMP-Launcher）与 `main`（meta 仓库）。
  **注意：merge 前先用 `git remote -v` / `git ls-remote` 确认上游默认分支名。**

## 3. 离线化设计（协议与改动全记录）

### 3.1 原版认证链路（已全部移除）

```
玩家 ──注册──> BeamMP论坛 ──获取key──> Launcher
Launcher ──pk──> auth.beammp.com/userlogin ──验证
Server 启动 ──AuthKey──> keymaster.beammp.com ──获取
Server 收到客户端 ──key+auth_key──> POST backend /pkToUser ──返回玩家身份
Server ──每5秒──> POST backend /heartbeat ──注册到公共服务器列表
Launcher ──> backend /servers-info ──服务器列表
Launcher ──> backend /builds/client ──下载客户端 mod（BeamMP.zip）
```

### 3.2 离线协议（现在的行为）

**握手协议不变**（版本包 `VC<ver>` → `A` → 身份包 → `P<id>` → `SR` 资源同步），
唯一变化是"身份包"的语义：

- **旧语义**：客户端发送"公钥"，服务端转发给 backend 验证换取用户名。
- **新语义**：客户端直接发送**玩家昵称**（UTF-8，≤32 字节，由玩家在游戏内自选，
  存在 Launcher 目录 `player_name` 文件）；服务端直接把它作为玩家名。
  空名字 → 服务端视为 `Guest`（受 `AllowGuests` 配置约束，默认允许）。

**所有互联网请求已移除**：
- Launcher：不再访问 auth/backend/forum 任何域名；更新检查、mod 下载、HTTP 代理
  全部禁用。mod 从本地获取（详见 3.4）。
- Server：不再做 keymaster/auth 验证；心跳线程只本地刷新信息包（供局域网 `I` 查询），
  不发 HTTP；`Application::CheckForUpdates()` 为空操作。
- 客户端 mod：不再通过本地代理拉取论坛头像。
- 保留的网络行为仅剩：游戏与 Launcher 的本机 127.0.0.1 通信、Launcher 与服务器的
  TCP/UDP 游戏数据连接（这就是联机本身）、Launcher 对用户输入 IP 的 DNS 解析。

### 3.3 代码改动明细（合并上游时可能冲突的位置）

#### BeamMP-Server-Offline
| 文件 | 改动 |
|---|---|
| `src/TNetwork.cpp` `TNetwork::Authentication()` | 删除 POST `/pkToUser` 的整段逻辑；身份包直接当作玩家名（sanitize：去控制字符、≤32 字节）；`SetName/ SetRoles("USER") / SetIsGuest()`。onPlayerAuth/postPlayerAuth Lua 事件保留。**上游客户端兼容**：若包内容是 ≥40 位纯 hex（官方 Launcher 发的是公钥），识别并命名 `Player-<前6位hex小写>`，官方玩家也能进离线服。 |
| `src/THeartbeatThread.cpp` `operator()` | 整个 HTTP 心跳循环替换为本地循环：每 5 秒调用 `GenerateCall()` 刷新 `lastCall`（供 `I` 信息包），状态置 Good，无任何网络请求。 |
| `src/Common.cpp` `Application::CheckForUpdates()` | 替换为空操作（状态置 Good）。 |
| `src/TConsole.cpp` `Command_NetTest()` | 不再请求 Server Check API，改为打印本地监听地址/玩家数。 |
| `src/TConfig.cpp` `FlushToFile()/ReadConfig()` | 删除 AuthKey 非空强制检查（离线版无需 AuthKey）；配置文件头注释改为离线版说明；AuthKey 字段保留但标注 unused。 |

#### BeamMP-Launcher-Offline
| 文件 | 改动 |
|---|---|
| `src/Security/Login.cpp` | **整文件重写**：`Login()` 不联网，解析 `{"username":"X"}` 存到 `player_name` 文件并返回 success；`CheckLocalKey()` 只读本地文件恢复昵称，恒 `LoginAuth=true`；新增 `SanitizePlayerName()`。 |
| `src/Startup.cpp` `CheckForUpdates()` | 空操作。 |
| `src/Startup.cpp` `PreGame()` | 不再从 backend 下载 mod。**优先级**：Launcher 同目录 `BeamMP.zip` 为权威离线副本（与游戏目录版本不同则覆盖安装）→ 游戏目录已有则直接用 → 都没有则 fatal 并提示从 Release 下载放置。官方版 mod 无法被静默装回（本 Launcher 永不联网）。 |
| `src/Network/Core.cpp` `Parse()` case 'B' | 服务器列表请求返回空数组 `B[]`（引导用户使用 Direct Connect）。 |
| `src/Network/Resources.cpp` `Auth()` | `TCPSend(PublicKey)` → `TCPSend(Username)`（≤32 字节）。 |
| `src/Network/Http.cpp` `StartProxy()` | 整个代理线程替换为空操作（`ProxyPort=0`）。 |
| `src/main.cpp` | 异常提示文案改为指向本仓库。 |

#### BeamMP-Offline（meta / 客户端 mod）
| 文件 | 改动 |
|---|---|
| `lua/ge/extensions/MPCoreNetwork.lua` `loginReceived()` | 跳过论坛头像的本地代理 HTTP 请求（`if false then ... end` 包裹，保留代码便于上游合并）。 |
| `ui/ui-vue/mods/BeamMP/views/BeamMPTOSView.vue` | **重写**：原 TOS 页（强制接受官方条款+外链）→ 离线欢迎页：说明离线特性 + 昵称输入框 + "Start Playing / Play as Guest" 按钮。仍写入 tosAccepted（保持路由守卫逻辑不变）。 |
| `ui/ui-vue/mods/BeamMP/views/BeamMPLoginView.vue` | **重写**：原账号登录页 → 玩家昵称设置页（Save Name / Play as Guest / Back）。 |
| `ui/ui-vue/mods/BeamMP/shared/beammpState.js` `login()` | 密码不再必需；只发 `{username}`。 |
| `ui/ui-vue/mods/BeamMP/layouts/BeamMPMain.vue` | 侧栏移除 Forum/Discord/Patreon/Docs 按钮；隐藏 Official/Featured/Partner 分类过滤；GitHub 按钮指向官方仓库。 |
| `.github/workflows/package.yml` | **新增**：把 `lua/ ui/ locales/ settings/ vehicles/ + LICENSE` 打包成 `BeamMP.zip`，push 到 main 产出 artifact，打 tag（`v*`）时附加到 Release。打包时**自动注入 `lua/ge/extensions/beammp/OFFLINE_BUILD.lua` 标记**（含 edition/version/commit/repository），用于识别离线版 mod（官方版 zip 没有此文件）。 |
| `ui/ui-vue/mods/BeamMP/index.js` | 主菜单按钮标题 `BeamMP` → `BeamMP Offline`。 |
| `ui/ui-vue/mods/BeamMP/layouts/BeamMPMain.vue` | 版本行加 `OFFLINE` 徽标（带 title 提示）。 |
| **v1.0.2 UI 大清理**（用户要求：只留直连+列表+身份，其余 UI+逻辑全删） | 删除：Patreon 横幅+图标、账号面板（论坛头像/角色徽章/ID/登出）→ 换成轻量 Player 徽章（名字+Edit→身份页）；死组件 BeamMPHome/TilesView/ModsCard/DirectConnectCard/PauseMainCard/PauseDisconnectModal 及路由/常量；暂停玩家列表的"打开论坛主页"按钮（保留复制名字）；死路由守卫分支与 logout()。增强：服务器浏览器空状态引导（含主线服务器需官方账号的提示）；直连页记忆上次 IP/端口 + 输入校验（内联错误提示）+ 当前身份显示；GitHub 链接指向本仓库。净删约 1100 行。 |
| `ui/ui-vue/mods/BeamMP/views/BeamMPTOSView.vue`（v1.0.2 补充） | 文案：昵称可随时在顶栏 Player 徽章或 Player Identity 页修改。 |

### 3.4 客户端 mod 分发方式（重要！）

离线版无法从 backend 下载 mod，因此：
1. CI（package.yml）从本仓库源码构建 `BeamMP.zip` 并发布到 GitHub Releases。
2. 玩家从 Release 下载 `BeamMP.zip`，放到**Launcher 同目录**或游戏目录
   `mods/multiplayer/` 下。
3. Launcher 的 `PreGame()` 会自动安装它（见 3.3）。

**mod zip 的目录结构** = 仓库里的 `lua/ ui/ locales/ settings/ vehicles/ icons/ scripts/`
目录直接打进 zip 根（+ LICENSE）。
UI（Vue）是**源码直接分发**的（BeamNG 运行时加载 `ui/ui-vue/mods/BeamMP/index.js`），无需 npm 构建。

**⚠ 发布检查清单（CI 已自动化校验，改 package.yml 时勿删）**：
zip 内必须存在 `scripts/BeamMP/modScript.lua` —— 它是 BeamNG 的 Lua 引导入口，
负责 `load()` 全部 MP 扩展（MPCoreNetwork/MPGameNetwork/MPVehicleGE/...）。
**丢失它 = UI 能显示但所有 MPCoreNetwork 等全局为 nil**
（症状：点击 BeamMP 按钮报 `attempt to index global 'MPCoreNetwork' (a nil value)`）。
v1.0.0-offline-mod 曾因漏打包 `scripts/` 和 `icons/` 触发此 bug，v1.0.1 修复。

历史备注：官方 zip 还含 `vehicles/unicycle/` 的 beamlings 宠物彩蛋（来自官方私有
Beamlings.zip，无开源许可信息），离线版不包含，不影响联机功能。

### 3.5 版本配套

三个组件必须配套使用（协议是自定义的）。对外版本号沿用上游 Launcher `2.8.1` 风格；
服务端 `ClientMinimumVersion` 校验依然有效（版本包协议未变）。

### 3.6 上游（官方）客户端兼容性 —— 语义说明

**问题**：官方（需登录账号的）Launcher/mod 能连我们的离线服务器吗？

**结论：能直接玩（天然支持），服务器无需额外改造即可接受官方客户端。** 原因与行为：
1. 握手协议（`VC<ver>` → `A` → 身份包 → `P<id>` → `SR`）与上游完全一致，未变。
2. 官方 Launcher 在"身份包"位置发送的是**账号公钥**（长 hex 字符串）；
   我们的 Server 识别该特征（≥40 位纯 hex），自动命名为 `Player-<前6位hex>`
   —— 官方玩家可读名字、正常进服、正常游戏。
3. 官方客户端在**有互联网**时才能完成它自己的登录/UI 流程（它连官方 auth）；
   连接目标则可以是我们任一离线服务器（游戏内 Direct Connect 输 IP）。
4. 官方客户端**无互联网**时无法通过官方登录流程（官方设计），此时只能换用
   本项目的离线 Launcher —— 这正是本项目存在的意义。
5. 语义对照表：

| 身份包内容 | 来源 | 服务器行为 |
|---|---|---|
| 普通昵称（≤32B） | BeamMP-Offline-Launcher | 直接作为玩家名 |
| 空 | 离线 Launcher（Guest 模式） | 命名 `Guest`（受 AllowGuests 约束） |
| ≥40 位纯 hex | 官方 Launcher（公钥） | 命名 `Player-<hex前6位>`，可正常游戏 |

### 3.7 mod 防覆盖 / 防被"更新"回官方版 —— 设计说明

**问题**：会不会有入口把玩家装好的离线 mod（BeamMP.zip）自动"更新"成官方版？

**结论：本项目组件链路中不存在任何在线更新路径；也无需修改 mod id。**
1. 我们的 Launcher **永不联网**、永不从 backend 下载/覆盖 mod（§3.3 Launcher 表）。
2. BeamNG 的 mod 管理器只对 BeamNG 官方 repo 的 mod 提供"更新"按钮；
   BeamMP.zip 不在 repo 上，游戏内没有更新入口。
3. BeamMP 本体从不经服务器下发（服务器只下发地图/车模资源）。
4. **不需要改 mod id/文件名**：`mods/multiplayer/BeamMP.zip` 的加载与
   `multiplayerbeammp` 激活 key 是 BeamNG 对该路径的特殊处理，改名有兼容风险
   且防不住唯一真正的覆盖源（见下）。防混淆通过"可见标识"达成：
   - UI 主菜单标题 `BeamMP Offline` + 侧栏 `OFFLINE` 徽标；
   - Launcher 控制台 OFFLINE 横幅；Server 配置头注释；
   - CI 注入的 `lua/ge/extensions/beammp/OFFLINE_BUILD.lua` 标记文件
     （官方版 zip 无此文件，可用于脚本校验真伪）。
5. **唯一覆盖途径 = 用户主动运行官方 Launcher**（它设计上就会用 backend 版
   BeamMP.zip 覆盖本地，且会清理 multiplayer 目录）。这无法从我们这边以代码
   阻止（那是官方程序的行为），处理方式：
   - Launcher/文档明确警告"不要混用官方 Launcher"；
   - 我们的 Launcher 把"Launcher 同目录的 BeamMP.zip"视为权威离线副本，
     启动时若与游戏目录版本不同则**重新覆盖安装**（一键修复被官方版覆盖的
     mod）。

## 4. 上游合并手册（保持与官方同步）

**原则**：上游的任何更新都值得合并进来；唯一要保护的是 3.3 表中列出的离线化改动。

标准流程（在每个离线仓库里执行）：

```bash
# 1. 一次性配置 upstream（每个仓库只需一次）
git remote add upstream https://github.com/BeamMP/<官方仓库名>.git

# 2. 拉取上游
git fetch upstream

# 3. 合并（meta 仓库上游分支可能是 main；Server/Launcher 是 master）
git merge upstream/minor        # Server 仓库上游默认分支是 minor

# 4. 解决冲突的判断标准：
#    - 若冲突发生在 3.3 表格所列的文件/函数：以"离线语义"为准手工融合，
#      但把上游新增的功能/修复一并保留。
#    - 若上游重构了认证相关代码（如删除/改名 Authentication、Auth 等）：
#      先看上游新实现，再把离线逻辑（本地昵称、无 backend）重新应用上去。
#    - 新增文件、无关文件：直接采用上游版本。
#    - 拿不准时：保留双方逻辑，跑通编译，再人工精简。
# 5. 编译冒烟（可选本地；否则交给 CI）
# 6. 提交合并，推送：
git push origin main             # 或 master，以仓库默认分支为准
```

**合并后自检清单**：
- [ ] `grep -rn "beammp.com" src/` —— Launcher/Server 中不应出现任何会实际发起的
      请求（注释除外）。UI 中只允许"打开浏览器"类按钮。
- [ ] Server `TNetwork::Authentication()` 无 HTTP 调用。
- [ ] Server 启动不要求 AuthKey 非空（TConfig.cpp）。
- [ ] Launcher `PreGame()` 不从网络下载 mod。
- [ ] CI 三个仓库全部绿灯。
- [ ] 打 tag 发布新 Release（meta 仓库 tag 会自动产出 BeamMP.zip）。

## 5. 构建（全部通过 GitHub Actions）

| 仓库 | Workflow | 产物 |
|---|---|---|
| BeamMP-Server-Offline | `.github/workflows/{linux,windows,release}.yml`（上游自带，已保留） | Windows/Linux Server 可执行文件 |
| BeamMP-Launcher-Offline | `.github/workflows/{cmake-linux,cmake-windows}.yml`（上游自带，已保留） | Launcher 可执行文件 |
| BeamMP-Offline | `.github/workflows/package.yml`（本项目新增） | `BeamMP.zip` 客户端 mod |

Server/Launcher 打 tag 会触发 release workflow 上传 Release。**发布一套完整版本时，
给三个仓库打同一个 tag**，然后把 Server/Launcher 产物 + BeamMP.zip 都放到
BeamMP-Offline 主仓库的 Release 里（或分别放在各自 Release）。

## 6. 用户使用流程（发布时的说明要点）

1. 下载 Release：Launcher（放游戏旁）、BeamMP.zip（放 Launcher 旁或游戏 mods/multiplayer/）、
   Server（主机运行）。
2. 主机：解压 Server → 运行 → 生成 ServerConfig.toml（无需 AuthKey）→ 配置端口/地图/人数。
3. 玩家：安装 mod zip → 启动 Launcher → 游戏内点 Multiplayer → 过欢迎页（输昵称）→
   Direct Connect 输入主机内网 IP:30814 → 进入。
4. 全程无需互联网，无需注册任何账号。

## 7. 新会话恢复方法（给未来的 AI/维护者）

如果这个对话丢失了，用类似下面的开场白即可无缝恢复：

> 我们在维护 BeamMP 的纯离线版（降低联机门槛，让无互联网用户内网联机，无需注册/账号/key）。
> 请先完整阅读 https://github.com/dksslq/BeamMP-Offline 仓库根目录的 CONTEXT.md（项目记忆）
> 和 UPSTREAM-SYNC.md（上游合并手册），相关仓库还有 dksslq/BeamMP-Server-Offline 和
> dksslq/BeamMP-Launcher-Offline（各自也有 CONTEXT.md）。之后请：保持三仓库与上游
> BeamMP 官方仓库的合并同步（保护离线化补丁，见各 CONTEXT.md 改动清单）、保持 GitHub
> Actions 自动构建与 Release 发布、按 CONTEXT.md §8 的方向继续改进离线体验。

**恢复后第一件事**：
1. `git clone` 三个仓库（或检查本地是否已有），读 CONTEXT.md 全文；
2. `git fetch upstream && git log HEAD..upstream/minor --oneline (Server; Launcher 用 upstream/master, meta 用 upstream/development)` 检查上游是否有新提交；
3. 按 §4 手册执行合并；
4. 检查 CI 状态（`gh run list` 或网页）。

## 8. 已知边界/未来可做的事

- `BeamMPPausePlayersCard.vue` 仍有玩家名 → 论坛个人页链接（点击打不开网页，无碍）。
- 主页（BeamMPHome.vue）的新闻卡片文案仍是官方社区口径，可改为离线欢迎内容。
- 可做：Launcher 端"最近连接服务器"历史列表（`server_history.json` + UI 展示），
  替代消失的公共服务器列表。
- 可做：Server 端 LAN 广播/自动发现（UDP broadcast），实现"不输 IP 的局域网发现"。
- 可做：Launcher `--player <name>` 命令行参数直接指定昵称。
- 服务器密码功能（上游有 Password 相关注释代码）可以启用，增强私服管控。
- 中文等其他语言的 UI 文案（locales/translations）可补充离线欢迎页翻译。
- modScript.lua 有游戏版本门禁（`compatibleVersion = 39`，即 BeamNG 0.39.x）。
  上游升级支持新版 BeamNG 时需同步此数字；版本不匹配时 mod 会自动停用并弹提示（属预期行为）。

## 9. 已修复的发布事故（供未来排查参考）

### 9.1 v1.0.0-offline-mod：zip 缺 scripts/ 导致 MPCoreNetwork 为 nil
- **症状**：进游戏点击 BeamMP 按钮报
  `attempt to index global 'MPCoreNetwork' (a nil value)`
  （位置：`guihooks.trigger("onBNGAPICallback", <id>, MPCoreNetwork.isLauncherConnected())`）。
- **根因**：package.yml 打包命令 `cp -r lua ui locales settings vehicles` 漏了
  `scripts/`（内含 BeamNG Lua 引导 modScript.lua）和 `icons/`（聊天/玩家列表图标）。
  没有 modScript.lua，BeamNG 不会 load() 任何 MP 扩展 → 全局为 nil。
  UI（Vue 源码）由游戏 UI 系统独立加载所以按钮仍可见 —— 症状与根因错位，具迷惑性。
- **修复**（v1.0.1-offline-mod）：打包加入 `icons scripts`；CI 增加
  "Verify critical files" 步骤校验 modScript.lua 等关键文件在 zip 内，缺失即 fail。
- **教训**：改打包脚本后，必须 `unzip -l` 人工核对与官方 zip 的文件清单差异
  （官方 zip = development 分支 + 私有 Beamlings.zip 覆盖物）。

## 10. 许可

所有代码遵循上游的 AGPL-3.0-or-later。离线化改动同样以 AGPL-3.0 发布。
本项目与 BeamMP 官方无隶属关系；请勿用于商业用途（遵循上游许可约束）。
