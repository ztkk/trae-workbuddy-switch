# 【Bug】Trae 网页登录报「未检测到【TraeWork CN】客户端」，但 Trae CN 明明装着

## 环境

- BuddySwitch `2026.10.141355`（Windows x64）
- 客户端：**只装了合并版** `%LOCALAPPDATA%\Programs\Trae CN\Trae CN.exe`
- `TRAE SOLO CN` / `TRAE SOLO` **都不存在**（客户端 2026-10-13 把 TraeWork 并入合并版，旧客户端已被卸载）
- 作用域：TraeWork 国内版（URL `/s/trae-cn/accounts`）

## 复现

1. 打开「TraeWork 国内版 → 账号」
2. 点「添加账号」→「OAuth 网页登录」
3. 点「**开始 Trae CN 网页登录**」
4. 报错：

```
未检测到【TraeWork CN】客户端：请先安装该客户端，或在「设置」里指定它的可执行文件路径，再重试。
```

「启动客户端」按钮同样报这句。

## 根因

**指纹是文案错位**：按钮写 `Trae CN`（那是**区域名**），报错写 `TraeWork CN`（那是**程序位名**）。

- 前端唯一持有的轴是**区域**（作用域写进 URL `/s/trae-cn/…`），`trae-oauth-login-dialog.tsx`
  用 `useTraeVariant()` 取值后原样传下去 ⇒ 后端收到的是 `"cn"`。
- 而 `trae_oauth_start` / `trae_launch_client` 收的是**程序位**级参数，
  解析走 `TraeVariant::parse`，其宽容别名把 `"cn"` 读成了 `TraeWork`
  （`crates/buddy-switch-core/src/modules/trae/variant.rs:307`）。
- 客户端合并之后 `TraeWork` 的候选（`TRAE SOLO CN.exe` / `TRAE SOLO.exe`）在本机不存在
  ⇒ `detect_install_for` 返回 `installed: false` ⇒ `data_dir_missing_reason` 给出「未检测到」。

顺带说明**按报错提示操作也救不了**：设置里的客户端路径是**按区域**存的
（`TraeSettings::trae_paths`，键 `cn`/`global`），而 `detect_install_for` 的采纳条件是
**变体级**的（`platform.rs` 要求 `variant_of_name(文件名) == variant`）。CN 区域三个程序位
共用一个路径槽，指向 `Trae CN.exe` 时只有合并版槽位认它 —— 所以填了路径，`TraeWork` 依然报错。

## 影响面

凡是 `TRAE SOLO CN` 被卸载/合并的机器（即所有跟进 2026-10-13 客户端升级的用户），
**国内版的 OAuth 网页登录与「启动客户端」完全不可用**。用户无法自救。

## 我的修法（已本地实现并验证，待您确认方向）

给这两个入口换一层入参解析 `variant_for_program_entry`，判据只看**入参有没有携带程序位信息**：

| 入参 | 处理 |
|:--|:--|
| 纯区域标识（`cn` / `global`） | 挑该区域**实际装着**的程序位（**合并版优先** —— 已合并机器上合并版与遗留 TraeCode 候选名逐字相同，探测都会说已安装） |
| 程序位标识（`trae_work` / `trae` / `trae_merged` …） | **原样尊重**，没装就响亮失败（静默替换会把登录态写进另一个客户端） |
| 缺失 / 无法识别 | 仍是老行为（回落默认变体） |

结果**绝不跨区域**。配套 `TraeRegion::is_region_token` 把「纯区域标识」与历史产品线标识分开，
`parse` 行为逐字不变（老链接继续可用）。前端一行不改。

本机实测：`"cn"` → `TraeMerged (Trae CN)`、已安装、数据目录 `%APPDATA%\Trae CN`。

## 想先请教两点

1. **方向**：`variant.rs` 整个模块已标 deprecated，我注意到 `.trellis/tasks/09-21-trae-region-program-model`
   有计划把 `TraeVariant` 收尾到 `(TraeRegion, TraeProgram)` 双轴。我的改动落在同一片区域，
   想先确认与您的规划是否冲突 —— 如果有，我按双轴口径改写。
2. **更彻底的做法**：也可以让前端显式传**程序位**（从 `get_trae_variants` 的
   `programs[].variant` 里挑当前区域已装的那条），后端只在「纯区域标识」时兜底。
   我选了后端落地，因为前端唯一持有的轴本来就是区域、且弹窗刻意只依赖 URL；
   但如果您认为前端显式传更符合本仓「不静默替换」的取向，我可以改。

需要的话我把补丁整理成 PR。
