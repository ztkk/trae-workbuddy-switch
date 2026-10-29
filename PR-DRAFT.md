# fix(trae): 区域标识不再被当成程序位，修已合并客户端上的登录失败

## 复现步骤

前置：机器上**只装了合并版** `Trae CN`（`TRAE SOLO CN` / `TRAE SOLO` 都不存在，
即客户端 2026-10-13 合并之后的常态）。

1. 打开「TraeWork 国内版 → 账号」（URL `/s/trae-cn/accounts`）
2. 「添加账号」→「OAuth 网页登录」→ 点「**开始 Trae CN 网页登录**」
3. 报错：`未检测到【TraeWork CN】客户端：请先安装该客户端，或在「设置」里指定它的可执行文件路径，再重试。`

「启动客户端」按钮同样报这句。

## 根因

**指纹是文案错位**：按钮写 `Trae CN`（**区域名**），报错写 `TraeWork CN`（**程序位名**）。

- 前端唯一持有的轴是**区域**（作用域写进 URL `/s/trae-cn/…`），
  `trae-oauth-login-dialog.tsx` 用 `useTraeVariant()` 取值后原样传给后端 ⇒ 后端收到 `"cn"`。
- 而这两个入口收的是**程序位**级参数，解析走 `TraeVariant::parse`，
  其宽容别名把 `"cn"` 读成了 `TraeWork`（`variant.rs:307`）。
- 客户端合并后 `TraeWork` 的候选（`TRAE SOLO CN.exe` / `TRAE SOLO.exe`）不存在
  ⇒ `detect_install_for` 返回 `installed: false` ⇒ `data_dir_missing_reason` 给出「未检测到」。

⚠️ **按报错提示去「设置」里指定路径也救不了**：设置里的客户端路径按**区域**存
（`trae_paths`，键 `cn`/`global`），而 `detect_install_for` 的采纳条件是**变体级**的。
CN 区域三个程序位共用一条路径槽，指向 `Trae CN.exe` 时只有合并版槽位认它。

## 改动

新增 `variant_for_program_entry` 作为 `trae_oauth_start` / `trae_launch_client`
（webui 与桌面两条通道各两处）的入参解析。判据只看**入参有没有携带程序位信息**：

| 入参 | 处理 | 理由 |
|:--|:--|:--|
| 纯区域标识（`cn` / `global`） | 挑该区域**实际装着**的程序位 | 区域标识不带程序位信息，只能由后端落地到一条真实客户端 |
| 程序位标识（`trae_work` / `trae` / `trae_merged` …） | **原样尊重**，没装就响亮失败 | 那是点名要操作哪条程序位；静默替换会把登录态写进另一个客户端 |
| 缺失 / 无法识别 | 回落默认变体 | 老调用点行为不变 |

**合并版优先**：已合并机器上合并版与遗留 TraeCode 候选名逐字相同、探测都会说已安装，
取首个装着的会落在遗留 TraeCode 上（界面上「探测到的是哪条产品线」会说错）。

结果**绝不跨区域**。配套 `TraeRegion::is_region_token` 把「纯区域标识」与历史产品线标识
分开；`TraeRegion::parse` 行为逐字不变（老链接/老前端继续可用）。**前端一行不改。**

## 验证

本机：只装合并版 `Trae CN`，`TRAE SOLO CN` 已卸载。

**真机探针**（`cargo test -p buddy-switch-core --test trae_variant_probe -- --ignored --nocapture`）：

```
入参 variant="cn"        → TraeMerged (Trae CN)   已安装=true  数据目录=…\Roaming\Trae CN
入参 variant="trae_work" → TraeWork (TraeWork CN) 已安装=false 数据目录=…\Roaming\TRAE SOLO CN
入参 variant="trae"      → Trae (TraeCode CN)     已安装=true  数据目录=…\Roaming\Trae CN
```

`"cn"` 落到真正装着的合并版；同时授权页参数也走对 —— 合并版 `packageType = TRAE_CN`
取的是 TRAE 那把 `client_id`，而不是原来 SOLO 那把。

**门禁**：

| 检查 | 结果 |
|:--|:--|
| `npm run build`（tsc + check:api / docker / store-selectors / scope / npm-platforms） | 通过 |
| `npm run check:api` | routes=182 / call=172 / server=166 / invoke=190 |
| `npm run check:scope` | 291 passed / 0 failed |
| `cargo test -p buddy-switch-core --lib` | 1258 passed，**3 failed**（见下，与本补丁无关） |
| 新增 3 条测试 | 全绿 |
| `cargo test -p buddy-switch-gateway` | 41 passed |
| `cargo test -p buddy-switch-server` | 49 passed |
| `cargo check --workspace --all-targets` | 0 warning |

## ⚠️ 三条红**与本补丁无关**（已在未打补丁的基线上验证）

我用 `git stash` 把本补丁摘掉后在**同一台机器**上复跑，这三条照样红：

1. `trae::account::tests::refresh_chain_uses_legacy_only_when_credential_missing`
2. `trae::oauth::tests::login_start_fails_loudly_when_device_identity_missing`

两者都断言 `data_dir_missing_reason(TraeWork)` 的「**装了但从未启动过** ⇒ 缺设备凭证」那一支，
而本机没装 TraeWork ⇒ 走的是「未检测到客户端」那一支。**装上 `TRAE SOLO CN` 即转绿**，
与解析改动无关（我没碰 `account.rs` / `oauth.rs`）。

3. `proxy_kpi_view::tests::stale_day_is_unknown_not_zero` —— **偶发**并行 flake
   （连跑 3 次红 1 次，单跑必过）。

## 待确认

`variant.rs` 整个模块已标 deprecated，`.trellis/tasks/09-21-trae-region-program-model`
有计划收尾到 `(TraeRegion, TraeProgram)` 双轴。我的改动落在同一片区域 —— 若与您的规划
冲突，我按双轴口径改写。另：也可改成前端显式传程序位、后端只在纯区域标识时兜底，
若那更符合本仓「不静默替换」的取向，我可以改。

🤖 Generated with [Claude Code](https://claude.com/claude-code)
