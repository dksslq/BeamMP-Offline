# Upstream Sync Guide / 上游合并操作手册

本仓库是 [BeamMP/BeamMP](https://github.com/BeamMP/BeamMP) 的离线化分支。
完整背景请先阅读 [CONTEXT.md](./CONTEXT.md)。

## 快速合并

```bash
git remote add upstream https://github.com/BeamMP/BeamMP.git   # 一次即可
git fetch upstream
git merge upstream/development  # meta 仓库上游默认分支是 development
```

## 本仓库的离线化补丁（合并冲突时必须保留的部分）

| 文件 | 离线改动 |
|---|---|
| `lua/ge/extensions/MPCoreNetwork.lua` | `loginReceived()` 跳过论坛头像代理请求 |
| `ui/ui-vue/mods/BeamMP/views/BeamMPTOSView.vue` | TOS 页 → 离线欢迎页（含昵称输入） |
| `ui/ui-vue/mods/BeamMP/views/BeamMPLoginView.vue` | 账号登录页 → 本地昵称设置页 |
| `ui/ui-vue/mods/BeamMP/shared/beammpState.js` | `login()` 无需密码 |
| `ui/ui-vue/mods/BeamMP/layouts/BeamMPMain.vue` | 移除 Forum/Discord/Patreon/Docs 侧栏链接、隐藏在线服务器分类 |
| `.github/workflows/package.yml` | 新增的 mod 打包 CI |

## 冲突处理原则

1. 上游对 mod 功能的更新（车辆同步、Lua API、UI 改进等）**全部接纳**；
2. 上面表格中的离线化语义**必须保留**；若上游重写了同一区域，把离线语义
   重新应用到上游的新实现上；
3. 上游若新增与在线服务相关的调用（新 backend 端点等）：不接入，保持离线；
4. 合并后运行自检清单（见 CONTEXT.md §4）。

## 冲突最少的技巧

上游对 `views/*.vue` 的重写概率较高。若 `BeamMPTOSView.vue` /
`BeamMPLoginView.vue` 冲突过大，可接受上游版本后**重新套用离线改造**
（照着 CONTEXT.md §3.3 的表格重做一遍这两个文件），这比手工解决大面积
Vue 冲突更快更稳。
