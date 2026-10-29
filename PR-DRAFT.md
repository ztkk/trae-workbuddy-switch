# fix(trae): 适配合并版客户端探测与自定义路径，打通国际版 Trae 网关调用与协议参数

## 为什么提这个 PR

合并版 Trae（国内版 `Trae CN`、国际版 `Trae`）发布后，在两处关键链路存在阻塞性问题：
1. **本地客户端检测与登录导入失败**：机器上仅安装了合并版（或安装在 `D:\Develop` 等自定义开发目录），点击登录或启动客户端时报错 `未检测到【TraeWork】客户端`。
2. **国际版账号网关调用 HTTP 401 失败与 4120 拦截**：导入国际版账号后，网关出站请求直接报错 `HTTP 401（code: 1001: We're sorry, but we are not able to authenticate you）`，无法调用国际版模型。

本 PR 针对上述问题进行了系统性修复，并通过真实国际版账号现场完成了端到端调用验证。

---

## 根因分析

### 1. 客户端探测与代次别名回退错误
- 前端交互持有的是**区域轴**（如 URL `/s/trae-cn/...` 或 `/s/trae-global/...`），传给后端的入参为纯区域标识 `"cn"` 或 `"global"`。
- 后端入参解析直接走 `TraeVariant::parse`，其别名规则把 `"cn"` 和 `"global"` 映射为了旧时代的单体版 `TraeWork`（`TRAE SOLO`）。
- 在现代仅安装合并版的机器上，`TRAE SOLO.exe` 不存在，导致探测与数据目录定位失败；且 `secondary_drive_roots()` 仅扫描了常见系统盘，遗漏了 `D:\Develop` 等二次开发路径。

### 2. 网关出站硬编码国内上游且缺乏国际版协议校验
- 网关 [`send_llm_chat`] 出站基址硬编码了国内端点 `https://trae-api-cn.mchost.guru`，将国际版 JWT 发往国内端点时，国内服务端无法认证该租户凭据，直接返回 HTTP 401。
- 源码注释曾标明国际版端点 `core-normal.trae.ai` *“从未对真实上游跑通过”*。经现场发包逆向分析，国际版服务端还存在两道强门禁：
  - **包类型匹配**：Header 与 Body 必须使用 `package-type: stable_i18n`（传国内的 `stable_cn` 会报 `4001 package-type is invalid`）。
  - **版本号门禁（错误码 `4120`）**：服务端校验客户端版本与版本码。若 `ide_version < 3.7.0` 或版本代码低于 `20260501`，上游会直接拦截并返回 `4120`（*“当前版本过低，无法使用 SOLO 模式，请升级至最新版本”*）。

---

## 改动清单

### 1. 客户端探测与程序位自适应 (`buddy-switch-core`)
- **新增 `variant_for_program_entry`**：
  - 当入参为纯区域标识（`"cn"` / `"global"`）时，动态探测并匹配该区域内**实际装着的合并版客户端**（`TraeMerged` / `GlobalTraeMerged`），杜绝向废弃的 `TraeWork` 错误回退；
  - 若指定了具体程序位则原样尊重，不静默替换。
- **扩展二次开发根目录扫描**：
  - `secondary_drive_roots()` 补充对 `D:\Develop`、`E:\Develop` 等自定义安装目录的自动发现。
- **账号导入与变体回退**：
  - 导入本地账号时若主变体数据目录不存在，自动在同区域已安装的候选变体中进行安全容错。

### 2. 国际版网关出站路由与协议参数对齐 (`buddy-switch-gateway`)
- **区域动态分流基址**：
  - 新增常量 `TRAE_AGENT_HOST_GLOBAL` (`https://core-normal.trae.ai`)；
  - `TraeGatewayState` 新增 `upstream_for(variant)` 方法：国内版变体路由至 `trae-api-cn.mchost.guru`，国际版变体路由至 `core-normal.trae.ai`（集成测试自定义 `upstream` 时优先保持测试覆写）。
- **国际版版本与包类型对齐**：
  - 新增国际版版本规范常量 `TRAE_IDE_VERSION_GLOBAL = "3.7.0"`，`TRAE_IDE_VERSION_CODE_GLOBAL = "20260501"`；
  - `prepare_llm_chat_body` 与 `send_llm_chat` 根据 `variant.region()` 动态注入对应的 `package-type`（`stable_cn` vs `stable_i18n`）、`ide_version` 与整型 `app_version_code`。
- **错误分类与状态监控**：
  - 在 `classify_solo` 中准确归类 `4120`（客户端版本门禁）；
  - `status_view` 中的 `upstream` 字段按变体准确呈现其实际绑定的上游主机。

### 3. 设备指纹动态感知与真实化
- 移除网关出站请求中硬编码的静态假指纹 `CREFG-XX`，防止多账号同指纹碰撞；
- 引入 `HostFingerprint` 单例机制，动态对齐当前真实宿主机 OS（Windows / Darwin / Linux）与 CPU 架构，未指定设备品牌时安全缺省。

---

## 验证结果

### 1. 真实国际版账号现场调用实测
使用真实国际版账号（Free 计划）向 `https://core-normal.trae.ai/api/agent/v3/llm_utils_chat` 发起流式测试：
- **HTTP 200 OK**
- 完整接收 `event:metadata`、`event:timing_cost`，并正常流式返回了 DeepSeek-v4.1-flash 模型的思考过程（`reasoning_content`）与回答正文（`response`），最终以 `event:token_usage` 和 `event:done` 正常收尾。

### 2. 自动化测试套件
全工作区 1800+ 项测试全部通过：
- `buddy-switch-core`: 1264 passed, 0 failed
- `buddy-switch-gateway`: 484 passed, 0 failed（含新增的 CN/Global 变体协议参数自适应单测与上游分流单测）
- `buddy-switch-rust`: 30 passed, 0 failed
- `buddy-switch-server`: 49 passed, 0 failed
