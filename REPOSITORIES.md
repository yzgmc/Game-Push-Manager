# Game Push Manager — 仓库功能总览

> 本文档面向**后续接手的 AI / 开发者**：仅阅读本文件即可掌握 3 个子仓库的全部功能、架构、协作流程与扩展方式，无需重新探索代码。
> 文档基于实际源码逐文件核对编写，最后更新与代码同步。
> 主仓库（Game-Push-Manager）远程：`https://github.com/yzgmc/Game-Push-Manager`

---

## 目录

- [一、项目总览](#一项目总览)
- [二、gpm-common（共享库）](#二gpm-common共享库)
- [三、gpm-web-admin（融合体：网页后台 + 服务端）](#三gpm-web-admin融合体网页后台--服务端)
- [四、gpm-client（Windows 客户端）](#四gpm-clientwindows-客户端)
- [五、跨仓库协作流程](#五跨仓库协作流程)
- [六、开发约定](#六开发约定)
- [七、扩展指南](#七扩展指南)
- [八、已知问题与落差](#八已知问题与落差)
- [九、给后续 AI 的开发指引](#九给后续-ai-的开发指引)

---

## 一、项目总览

### 1.1 项目定位

Game Push Manager（GPM）是一套**游戏整合包 / 模组推送分发与监控管理系统**，核心场景：

- 管理员通过网页后台上传整合包 / 模组，管理用户、查看各端运行状态。
- 融合体服务端对外提供同步 / 下载 API，同时接收各端心跳并聚合展示仪表盘 + 状态指示灯。
- Windows 客户端从服务端拉取条目，下载整合包后**自动安装模组加载器**并一键启动游戏。
- 所有端通过 **Push 模型**主动向融合体后台上报心跳。

当前原生支持 **Minecraft**，通过 `gpm-common` 的游戏适配器机制可扩展其他游戏。

> 历史上有第 4 个仓库 `gpm-server`（面向 Windows 的独立服务端 + PySide6 桌面 GUI），但其 Web API、存储、上报、网页管理 UI 均被 `gpm-web-admin` 完全覆盖且后者更先进（多角色权限、NeoForge、模组-整合包关联、仪表盘、系统更新）。`gpm-server` 唯一独有的是 PySide6 桌面 GUI 与 Nuitka Windows EXE 一键部署，已于本次从主仓库移除。如需 Windows 桌面 GUI 部署形态，可基于 `gpm-web-admin` 自行接 PySide6 壳，或参考 git 历史恢复 `gpm-server`。

### 1.2 三个仓库关系

```
                        ┌─────────────────────────────┐
                        │      gpm-common（共享库）      │
                        │  数据模型 / 协议 / 适配器 /     │
                        │  心跳 / 认证 / 灯色 / 存储      │
                        └──────────────┬──────────────┘
                                       │ pip install -e .
                          ┌────────────┴────────────┐
                          │                         │
           ┌──────────────▼──────────────┐  ┌───────▼────────────┐
           │   gpm-web-admin             │  │   gpm-client       │
           │ （融合体：网页后台 + 服务端）  │  │ （Windows 客户端）  │
           │ FastAPI + 静态前端          │  │ PySide6 桌面应用    │
           └──────────────┬──────────────┘  └───────┬────────────┘
                          │                         │
                          └────────────┬────────────┘
                                       │ POST /api/v1/report（Heartbeat）
                                       ▼
                              gpm-web-admin 内存 ReportStore
                              → 仪表盘聚合 + 灯色
```

**关键设计：融合体**

`gpm-web-admin` 是"网页后台 + 网页服务端"的**单进程融合体**：同时承担
1. 接收各端心跳的后台角色（`/api/v1/report`、`/api/v1/dashboard`）；
2. 提供上传 / 同步 / 下载 / 用户管理 / 自动更新的服务端角色。

启动后默认把 `admin_url` 指向自己，**自己上报给自己**，自动纳入仪表盘。

### 1.3 技术栈

| 仓库 | 语言 | 框架 | GUI | 打包 |
|------|------|------|-----|------|
| gpm-common | Python ≥3.9 | Pydantic v2 + httpx（可选 fastapi） | — | pip 包 |
| gpm-web-admin | Python ≥3.9 | FastAPI + uvicorn | 纯静态 HTML/JS | systemd / Nuitka |
| gpm-client | Python ≥3.9 | httpx | PySide6 | Nuitka 单文件 exe |

### 1.4 协议版本

`API_PREFIX = "/api/v1"`、`API_VERSION = "1.0.0"`（见 [protocol.py](file:///workspace/gpm-common/gpm_common/protocol.py)）。所有端必须使用相同协议版本。

---

## 二、gpm-common（共享库）

### 2.1 定位

所有端共用的数据契约、协议常量、游戏适配器、心跳 / 认证 / 灯色 / 存储工具。**零业务依赖**，仅 `pydantic` + `httpx`（fastapi 局部导入，可选）。

- 包名：`gpm-common`，版本 `1.0.0`
- 安装：`pip install -e .` 或 `pip install git+https://github.com/yzgmc/gpm-common.git`
- GitHub：`https://github.com/yzgmc/gpm-common`

### 2.2 目录结构

```
gpm-common/
├── gpm_common/
│   ├── __init__.py          # 聚合导出所有公开符号
│   ├── models.py            # Pydantic 数据模型
│   ├── protocol.py          # API 路径/版本/错误码/GamePushError
│   ├── game_adapter.py      # GameAdapter 抽象基类 + Registry
│   ├── adapters/
│   │   ├── __init__.py      # register_builtin_adapters() 注册 Minecraft
│   │   └── minecraft.py     # MinecraftAdapter 完整实现
│   ├── heartbeat.py         # Heartbeat 载荷模型
│   ├── reporter.py          # Reporter 后台线程
│   ├── storage.py           # 文件存储路径/防穿越/目录大小
│   ├── hashing.py           # sha256 工具
│   ├── auth.py              # pbkdf2 密码 + HS256 JWT + FastAPI 依赖
│   └── lights.py            # 状态指示灯（绿/黄/红/灰）
├── setup.py
├── requirements.txt
├── gpm-deploy.sh            # 一键部署融合体到 Linux 服务器
└── README.md
```

### 2.3 模块详解

#### models.py（[查看](file:///workspace/gpm-common/gpm_common/models.py)）

Pydantic v2 模型，"基类—创建—完整"三层结构：

| 模型 | 字段要点 |
|------|---------|
| `GameInfo` | `name` / `display_name` / `adapter` / `enabled` |
| `LaunchConfig` | `java_path` / `jvm_args` / `extra_args` / `username?` / `uuid?` / `access_token?` / `user_type`（默认 `offline`，可选 `msa`）/ `game_dir?`（HMCL 风格版本隔离目录，None=与 install_dir 共享） |
| `ModpackBase` | `name` / `version` / `game` / `game_version` / `mod_loader`（默认 `vanilla`）/ `mod_loader_version` / `description` / `enabled` |
| `ModpackCreate` | 继承 ModpackBase（上传用） |
| `Modpack` | 继承 ModpackBase + `id` / `file_name` / `file_size` / `file_hash` / `created_at` / `updated_at` |
| `ModBase` | `name` / `version` / `game` / `game_version` / `mod_loader` / `mod_loader_version` / `modpack_id` / `description` / `enabled` |
| `ModCreate` / `Mod` | 同 Modpack 模式 |
| `SyncResponse` | `protocol_version` / `server_name` / `modpacks` / `mods` / `games` / `server_time` |
| `StatusResponse` | `server_name` / `server_kind` / `status` / `protocol_version` / `uptime_seconds` / `modpack_count` / `mod_count` / `storage_used_bytes` / `started_at` |

> ⚠️ `mod_loader` 字段注释仅写 `vanilla/forge/fabric/quilt`，但 `MinecraftAdapter.supported_mod_loaders()` 实际返回 5 项（含 `neoforge`）。模型本身不限制取值。

> ℹ️ `LaunchConfig` 已扩展支持正版账号信息（`username`/`uuid`/`access_token`/`user_type`）与版本隔离（`game_dir`），由 gpm-client 的 MSA 登录与版本管理功能消费，详见 [四、gpm-client](#四gpm-clientwindows-客户端)。

#### protocol.py（[查看](file:///workspace/gpm-common/gpm_common/protocol.py)）

- `API_PREFIX = "/api/v1"`、`API_VERSION = "1.0.0"`
- `route(path)`：拼接完整路由（自动补 `/` 前缀）
- `ErrorCode`：`UNKNOWN / NOT_FOUND / VALIDATION_ERROR / FILE_TOO_LARGE / HASH_MISMATCH / ADAPTER_NOT_FOUND / PROTOCOL_MISMATCH / INTERNAL_ERROR / UNAUTHORIZED`
- `GamePushError(Exception)`：`message` / `code` / `status_code` / `details` / `to_dict()` → `{"error":{"code","message","details"}}`

#### game_adapter.py（[查看](file:///workspace/gpm-common/gpm_common/game_adapter.py)）

- `GameAdapter(abc.ABC)`：类属性 `game_name` / `display_name`
  - 抽象方法：`game_info()` / `validate_modpack(archive_path)` / `build_launch_command(install_dir, launch_config, modpack_meta)`
  - 默认实现：`validate_mod()`（存在且非空）/ `detect_metadata()`（返回 None）/ `install_dir_hint()`（`<base>/<game>/<name>`）/ `supported_mod_loaders()`（返回 `[]`）
- `GameAdapterRegistry`：`register(adapter)` / `get(name)` / `require(name)`（不存在抛 404）/ `all_games()` / `clear()`
- `adapters/__init__.py` 导入即调用 `register_builtin_adapters()` 注册 `MinecraftAdapter`

#### adapters/minecraft.py（[查看](file:///workspace/gpm-common/gpm_common/adapters/minecraft.py)）— 重点

`MinecraftAdapter`，`game_name="minecraft"`：

- `supported_mod_loaders()` → `["vanilla", "forge", "neoforge", "fabric", "quilt"]`（**5 种**）
- `validate_modpack()`：zip 合法 + 含 `manifest.json` / `modrinth.index.json` / `overrides/` / 任意 `.jar` 即通过
- `detect_metadata(archive_path)`：
  - **CurseForge**（`manifest.json`）：`minecraft.version` → game_version；`minecraft.modLoaders` 中 `primary=true` 项的 `id`（如 `fabric-loader-0.15.7`）经 `_parse_cf_loader_id` 解析为 loader + version
  - **Modrinth**（`modrinth.index.json`）：`dependencies.minecraft` → game_version；按 `fabric-loader/fabric/quilt-loader/quilt/neoforge/forge` 顺序识别 loader
  - 裸整合包 / 异常 → None
- `_parse_cf_loader_id()`：**更长前缀优先**（避免 neoforge 被误识别为 forge）
- `detect_mod_metadata(jar_path)`：从模组 jar 解析
  - Fabric（`fabric.mod.json`）：`depends.fabric` → fabric
  - Quilt（`quilt.mod.json`）：`quilt_loader.depends.quilt_loader` → quilt
  - NeoForge/Forge TOML（`META-INF/neoforge.mods.toml` 优先，否则 `META-INF/mods.toml`）：文件名含 neoforge → neoforge，否则 forge
  - Forge 1.12-（`mcmod.info`）：`mcversion` → game_version
- `build_launch_command(install_dir, launch_config, modpack_meta)`（**已重写为完整 Mojang 版本 JSON 启动器**）：
  1. 定位版本 JSON：优先 `modpack_meta["version_id"]` 显式指定，否则 `_resolve_version_id` 按加载器类型推断（prefixes 字典覆盖 fabric/quilt/forge/neoforge/vanilla 五种）
  2. 无版本 JSON 回退 `_legacy_launch_command`：保留旧 `-jar` 方式（vanilla/forge/neoforge 走 `-jar <jar> --gameDir`；fabric/quilt 走 `-jar <loader_jar> --gameDir --gameVersion`），兼容裸目录。**注意：legacy 路径不消费账号信息。**
  3. `_merge_inherited`：递归合并 `inheritsFrom` 链（libraries/arguments 追加，mainClass/assetIndex/minecraftArguments 子覆盖父）
  4. `_build_classpath`：组装 classpath（libraries + **原版 client jar 必须在 classpath**，否则 Fabric/Quilt game provider 报 "couldn't locate the game!"），原版 jar 由 `_find_vanilla_client_jar` 沿 inheritsFrom 链查找
  5. `_extract_natives`：解压 LWJGL 等 native 库（.dll/.so/.dylib）到 `versions/natives`
  6. 取 `mainClass`（默认 `net.minecraft.client.main.Main`）
  7. 组装 `[java, *jvm_args, -Djava.library.path, -cp, classpath, mainClass, *game_args, *extra_args]`
  8. `_expand_game_args`：展开 `arguments.game`（列表含 rules）或旧版 `minecraftArguments`（空格字符串）；账号信息从 `launch_config` 取——`username`/`uuid`/`access_token` 三者齐全 → 正版（msa）模式填充真实账号；否则离线模式（固定 `Player` 名 + uuid5 + 占位 token `"0"`）；`${game_directory}` 用 `launch_config.game_dir or install_dir`（透传版本隔离目录）
- jar 定位辅助：`_find_forge_jar` / `_find_neoforge_jar` / `_find_fabric_loader_jar` / `_find_vanilla_client_jar`（沿 inheritsFrom 链找根版本 → game_version 直接定位 → 兜底扫描 versions 目录）
- `_detect_java()`：依次 `which(java.exe / javaw.exe / java)`，兜底 `"java"`

#### heartbeat.py（[查看](file:///workspace/gpm-common/gpm_common/heartbeat.py)）

`Heartbeat(BaseModel)`：
- `reporter_id` / `kind`（`web-server|client`）/ `name` / `username?`（客户端登录后携带，后台按用户去重）/ `base_url?` / `status` / `protocol_version` / `sent_at` / `light?: Light` / `metrics: dict` / `extra: dict`

#### reporter.py（[查看](file:///workspace/gpm-common/gpm_common/reporter.py)）

`Reporter` 后台守护线程：
- 构造：`Reporter(admin_url, build_payload: Callable[[], Heartbeat], interval=10.0, timeout=5.0)`
- `start()`：daemon 线程 `name="gpm-reporter"`，**启动立即上报一次**
- `_loop()`：`_stop.wait(interval)` 等待间隔（被 set 提前退出），否则 `report_once()`
- `report_once()`：`httpx.Client` POST `{admin_url}/api/v1/report`，json=`payload.model_dump(mode="json")`；失败仅 warning 不影响主业务
- `stop()`：`_stop.set()` + `join(timeout=2)`

#### storage.py（[查看](file:///workspace/gpm-common/gpm_common/storage.py)）

目录约定：`<root>/{modpacks|mods}/<id>/{meta.json, <file_name>}`

- `build_storage_path(root, kind, id, file_name)` / `build_meta_path(root, kind, id)`
- `safe_join(base, *parts)`：防路径穿越（realpath 比较）
- `dir_size(path)`：递归累加文件大小
- `ensure_dir(path)` / `_sanitize(name)` / `_sanitize_filename(name)`

#### hashing.py（[查看](file:///workspace/gpm-common/gpm_common/hashing.py)）

- `compute_sha256(file_path, chunk_size=1<<20)`：流式 1MB 块
- `compute_bytes_sha256(data)` / `verify_file(file_path, expected_hash)`

#### auth.py（[查看](file:///workspace/gpm-common/gpm_common/auth.py)）

零外部依赖（仅标准库）：
- `hash_password(password, iterations=200_000)` → `pbkdf2_sha256$<iter>$<salt_b64>$<hash_b64>`
- `verify_password(password, stored)`：`hmac.compare_digest` 恒定时间比较
- `create_token(payload, secret, expires_seconds=86400)` / `decode_token(token, secret) -> TokenPayload`：HS256 JWT
- `require_token(secret)`：返回 FastAPI 依赖，仅校验 token 有效（任何登录用户通过）（**局部导入 fastapi.Header**）
- `require_admin(secret)`：在 `require_token` 基础上额外校验 `payload.role == "admin"`，非管理员抛 `AuthError("需要管理员权限", 403)` —— **gpm-web-admin 的写操作 / 仪表盘 / 系统更新均用此依赖**
- `generate_secret()`：`secrets.token_urlsafe(48)`

#### lights.py（[查看](file:///workspace/gpm-common/gpm_common/lights.py)）

- `LightLevel`：`GREEN/YELLOW/RED/OFF` + `priority()`（GREEN=0 < OFF=1 < YELLOW=2 < RED=3）
- `Light(BaseModel)`：`level` / `reason` / `checked_at`
- 默认阈值（环境变量可覆盖）：`GPM_LIGHT_DISK_YELLOW=85` / `GPM_LIGHT_DISK_RED=95` / `GPM_LIGHT_ERROR_RED=10`
- `compute_server_light(data_dir, error_count=0, ...)`：按严重度判定
  1. data_dir 非目录 → RED
  2. 磁盘占用 ≥ red → RED
  3. error_count ≥ err_red → RED
  4. 磁盘占用 ≥ yellow → YELLOW
  5. 其余 → GREEN
- `aggregate_light(levels)`：取最严重者（空列表 → OFF）

### 2.4 gpm-deploy.sh（[查看](file:///workspace/gpm-common/gpm-deploy.sh)）

一键部署"融合体"到 Linux 云服务器（root 执行）：
- 克隆 `gpm-common` + `gpm-web-admin` 到 `/opt/gpm`
- 创建 venv + 安装依赖
- 生成固定 `GPM_AUTH_SECRET` 写 `/opt/gpm/.auth_secret`
- 注册 systemd 服务 `gpm-web-admin.service`（端口 8001，`Restart=always`）
- 健康检查 + 登录验证 + 仪表盘自上报验证
- 用法：`sudo bash gpm-deploy.sh ghp_<token>`

---

## 三、gpm-web-admin（融合体：网页后台 + 服务端）

### 3.1 定位

**单进程融合体**：同时是网页后台（接收心跳 + 仪表盘）+ 网页服务端（上传/同步/管理）。启动后 `admin_url` 默认自指向，自己上报给自己。是当前唯一的 Web 服务端实现。

- GitHub：`https://github.com/yzgmc/gpm-web-admin`
- 默认端口 `8001`（`gpm-deploy.sh` 部署端口）

### 3.2 目录结构

```
gpm-web-admin/
├── run.py
├── app/
│   ├── main.py              # FastAPI app（路由分两组：后台 + 服务端）
│   ├── config.py            # Settings（含 sync_admin_users）
│   ├── storage.py           # 整合包/模组存储
│   ├── server_info.py       # 运行时统计
│   ├── reporter.py          # 自上报心跳
│   ├── report_store.py      # 内存心跳存储 + dashboard 聚合
│   ├── updater.py           # git 自动更新
│   └── routes/
│       ├── report.py        # POST /api/v1/report（接收心跳）
│       ├── dashboard.py     # GET /api/v1/dashboard（仪表盘）
│       ├── auth.py / config.py / games.py / status.py / sync.py
│       ├── modpacks.py / mods.py   # 服务端 CRUD
│       └── update.py        # 系统更新管理
├── static/
│   ├── index.html           # 仪表盘首页
│   ├── app.js               # 仪表盘前端（5s 轮询）
│   ├── admin.html           # 服务端管理 UI（5 Tab）
│   ├── admin.js             # 管理前端（含角色权限 UI）
│   ├── login.html
│   └── *.css
├── requirements.txt
└── README.md
```

### 3.3 入口与配置

`app/main.py`（[查看](file:///workspace/gpm-web-admin/app/main.py)）路由分两组：
```
# 后台路由
report.router     # POST /api/v1/report（无鉴权）
dashboard.router  # GET  /api/v1/dashboard（需 admin）
# 服务端路由
auth / config / games / status / sync / modpacks / mods / update
```
- 页面：`GET /` → index.html（仪表盘）、`/login`、`/admin`（admin 页面本身无鉴权，由前端 role 控制可见性 + 后端 API `require_admin` 拦截）
- `GET /api/info`：`service="gpm-web-admin-fusion"`、`kind`、`reporting_to`、`dashboard`、`server_name`
- startup：`start_reporter()` + `start_updater()`；shutdown：`stop_reporter()` + `stop_updater()`

Settings 字段：
- `server_kind = "web-server"`（固定）
- `port` 默认 `8001`
- `admin_url` 默认 `http://127.0.0.1:{port}`（自指向）
- `stale_seconds` 默认 `30`（离线判定阈值）
- 新增 `sync_admin_users(admin_users, source)`：合并其他服务端上报的管理员（带 `_source` 标记，source 不再上报自动清除）

**环境变量**（与历史 gpm-server 一致，便于环境复用）：

| 变量 | 默认 | 说明 |
|------|------|------|
| `GPM_HOST` | `0.0.0.0` | 监听地址 |
| `GPM_PORT` | `8001` | 监听端口 |
| `GPM_DATA_DIR` | 仓库根 / exe 同级 `data/` | 数据目录 |
| `GPM_SERVER_NAME` | `gpm-web-admin` | 服务名 |
| `GPM_MAX_UPLOAD_MB` | `4096` | 上传上限 |
| `GPM_AUTH_SECRET` | 随机 | JWT secret（未设则重启失效） |
| `GPM_ADMIN_URL` | `http://127.0.0.1:<port>` | 后台地址（默认自指向） |
| `GPM_PUBLIC_BASE_URL` | `http://127.0.0.1:<port>` | 上报可访问地址 |
| `GPM_REPORTER_INTERVAL` | `10` | 上报间隔秒 |
| `GPM_REPORTER_ID` | 自动生成 | 上报端 ID |
| `GPM_USERS` | — | `user:hash[:role],...` 只读用户表 |
| `GPM_LIGHT_DISK_YELLOW/RED/ERROR_RED` | 85/95/10 | 灯色阈值 |

### 3.4 路由清单（前缀 `/api/v1`）

| 方法 | 路径 | 鉴权 | 说明 |
|------|------|------|------|
| POST | `/auth/login` | 无 | 登录返回 token（24h）+ role |
| PUT | `/auth/password` | token | 改自己密码 |
| GET | `/users` | admin | 用户列表 |
| POST | `/users` | admin | 添加用户 |
| PATCH | `/users/{username}` | admin | 改角色 |
| DELETE | `/users/{username}` | admin | 删用户（至少留一个 admin） |
| GET | `/modpacks` | 无 | 整合包列表 |
| POST | `/modpacks` | admin | 上传（自动识别 loader/version） |
| GET | `/modpacks/{id}` | 无 | 详情 |
| PATCH | `/modpacks/{id}` | admin | 改元数据/上下架 |
| GET | `/modpacks/{id}/download` | 无 | 下载文件 |
| DELETE | `/modpacks/{id}` | admin | 删除 |
| GET/POST/PATCH/DELETE | `/mods/...` | 同上 | 模组 CRUD |
| GET | `/sync` | 无 | 客户端同步（仅 enabled） |
| GET | `/status` | 无 | 服务端状态 |
| GET | `/games` | 无 | 游戏列表 |
| GET | `/config` | 无 | 读运行时配置 |
| PUT | `/config` | admin | 改配置 + 热重启 reporter |
| POST | `/report` | 无 | 接收各端心跳上报 |
| GET | `/dashboard` | admin | 仪表盘聚合数据 |
| GET | `/update/status` | admin | 自动更新状态 |
| POST | `/update/check` | admin | 检查更新 |
| POST | `/update/apply` | admin | 立即应用更新 |
| PATCH | `/update/auto` | admin | 切换自动更新 |

### 3.5 存储结构

```
data/
├── users.json              # {username: {hash, role, _source?}}
├── server.json             # 运行时配置（admin_url 等）
├── .reporter_id            # 持久化上报 ID
├── modpacks/<id>/{meta.json, <原文件名>}
└── mods/<id>/{meta.json, <原文件名>}
```

### 3.6 mod_loader 自动识别链路（[查看](file:///workspace/gpm-web-admin/app/routes/modpacks.py)）

上传时表单 `mod_loader` 默认 `"auto"`，`game_version` 默认空。触发条件 `not game_version or mod_loader in ("", "auto")`：
1. `adapter = GameAdapterRegistry.get(game)`（`.get()` 不抛错）
2. 调 `adapter.detect_metadata(tmp_path)`（modpack）或 `detect_mod_metadata`（mod）
3. 用识别结果填回 `game_version` / `mod_loader`（**含 neoforge**）/ `mod_loader_version`
4. `except Exception: pass` 容错，识别失败不阻断上传
5. 返回值附 `auto_detected: bool`

> 服务端本身不写识别逻辑，**完全委托给 gpm_common 适配器**。

### 3.7 心跳上报（[查看](file:///workspace/gpm-web-admin/app/reporter.py)）

- 启动条件：`settings.admin_url` 非空（默认自指向，故默认启动）
- `_build_heartbeat()`：含 modpacks/mods 完整列表 + `admin_users`（含 hash，供后台同步）+ `light`（`compute_server_light`）
- `PUT /api/v1/config` 后 `restart_reporter()` 热生效

### 3.8 ReportStore 聚合逻辑（[查看](file:///workspace/gpm-web-admin/app/report_store.py)）— 核心

Push 模型，后台不轮询：
- `StoredReport`：单条上报记录，`is_stale()` = `now - last_seen_at > stale_seconds`
- `light_level()`：**离线强制 RED**；未上报 light → OFF；否则取上报 light.level
- `record(hb)`：按 `_dedup_key` 去重（客户端带 username 按 `user:{username}`，其余按 `reporter_id`）；锁外调 `sync_admin_users`
- `prune_stale()`：清理超过 `stale_seconds * 10` 的条目
- `aggregate()`：
  1. prune_stale
  2. 按 kind 分组生成 `by_kind`
  3. 推送条目聚合：遍历在线端 `metrics.modpacks/mods`，按 game 分桶，key=`{name}@{version}`
  4. `overall_light = aggregate_light([r.light_level() ...])`
  5. `kind_lights[kind] = aggregate_light(items.light.level)`
  6. 返回 `{generated_at, model:"push", reporters_total, reporters_online, overall_light, kind_lights, by_kind, games}`

### 3.9 自动更新机制（[查看](file:///workspace/gpm-web-admin/app/updater.py)）

基于 git，部署目录自动检测（`_CODE_DIR` / `_DEPLOY_DIR/gpm-common`）：
- `_DEFAULT_INTERVAL = 300.0`（5 分钟）
- `check_for_updates()`：`git fetch origin main`，比较 local SHA (HEAD) 与 remote SHA (origin/main)
- `apply_update()`：
  1. `git pull origin main`（先 gpm-common，再 gpm-web-admin）
  2. `pip install --quiet <gpm-common>`
  3. `pip install -r requirements.txt`（超时不阻断）
  4. `update_count += 1`
  5. 守护线程 `sleep(1.0)` 后 `os._exit(0)`（让 systemd `Restart=always` 拉起新版本）
- `_auto_check_loop()`：每 5 分钟检查，`auto_enabled` 且非打包环境则自动应用
- 打包环境（`_IS_COMPILED`）禁用 git 更新

**更新路由**（[查看](file:///workspace/gpm-web-admin/app/routes/update.py)，需 admin）：
- `GET /update/status`、`POST /update/check`、`POST /update/apply`、`PATCH /update/auto`

### 3.10 普通用户只读管理视图（双层防护）

**后端**：通过 `require_admin` 依赖拦截——仪表盘 `/dashboard`、系统更新 `/update/*`、用户管理 `/users`（POST/PATCH/DELETE）、所有写操作（POST/PATCH/DELETE modpacks/mods、PUT config）均需 admin role，普通用户调这些接口返回 403。读操作（list/detail/download/sync/status/games/config 读）开放。

**前端**：[login.html](file:///workspace/gpm-web-admin/static/login.html) 登录成功后把 `gpm_role` 存入 localStorage 并统一跳 `/admin`；[admin.js](file:///workspace/gpm-web-admin/static/admin.js) 用 `_isAdmin = localStorage.getItem(ROLE_KEY) === 'admin'` 控制 UI：
- 非管理员隐藏「返回仪表盘」链接、隐藏 `users`/`config`/`update` 三个管理类 Tab、隐藏所有上传表单
- 整合包/模组列表不渲染编辑/上下架/删除按钮（只保留下载）
- `loadUpdateStatus()` 及其 30s 轮询包在 `if (_isAdmin)` 内，普通用户不触发对 `/api/v1/update/*` 的请求

即使前端被绕过，后端 `require_admin` 仍拦截。

### 3.11 前端结构

**index.html + app.js**（仪表盘，[查看](file:///workspace/gpm-web-admin/static/app.js)）：
- `API = '/api/v1/dashboard'`，5 秒轮询
- 总体灯色区 + 摘要卡（在线数/整合包数/模组数）+ 灯色图例
- 三区块：服务端列表（`web-server`）、客户端列表（`client`）、游戏推送条目
- 卡片左边框按 `light.level` 上色

**admin.html + admin.js**（服务端管理，[查看](file:///workspace/gpm-web-admin/static/admin.html)）：
- 5 Tab：整合包管理 / 模组管理 / 用户管理 / 配置 / 系统更新
- **加载器下拉含 6 项**：`auto / vanilla / forge / neoforge / fabric / quilt`（确认含 neoforge）
- 模组表单含「所属整合包」下拉，模组表格多一列「所属整合包」
- 上传用 XMLHttpRequest 实现 `upload.onprogress` 进度回调
- `setInterval(loadStatus, 10000)` + `setInterval(loadUpdateStatus, 30000)`（后者仅 admin 触发）

---

## 四、gpm-client（Windows 客户端）

### 4.1 定位

Windows 桌面客户端（PySide6），从融合体服务端同步条目，下载整合包后**自动安装模组加载器**并启动游戏。已升级为**完整的正版 Minecraft 启动器**：HMCL 风格多版本隔离 + 微软账号正版登录 + Java 运行时自动安装 + 原版 MC 文件下载 + 优化启动命令。

- GitHub：`https://github.com/yzgmc/gpm-client`

### 4.2 目录结构

```
gpm-client/
├── run.py
├── app/
│   ├── main.py             # 入口（登录校验 + 启动 MainWindow + start_reporter）
│   ├── config.py           # ClientConfig + installed.json + msa_credentials
│   ├── api_client.py       # 服务端 HTTP 客户端
│   ├── sync_manager.py     # 同步 + 本地状态
│   ├── downloader.py       # 多线程分块下载 + sha256 校验
│   ├── installer.py        # 整合包解压 + 模组复制
│   ├── launcher.py         # 通过适配器启动游戏（含账号 + 版本隔离 + 自动内存）
│   ├── loader_installer.py # Fabric/Forge/NeoForge/Quilt 自动安装
│   ├── minecraft_installer.py  # 原版 MC 文件下载（BMCLAPI 镜像）— 新增
│   ├── java_installer.py   # Java 运行时自动安装（TUNA/Adoptium）— 新增
│   ├── msa_auth.py         # 微软账号正版登录（MSA→XBL→XSTS→MC）— 新增
│   ├── version_manager.py  # HMCL 风格多版本隔离 — 新增
│   ├── reporter.py         # 心跳上报
│   └── ui/
│       ├── main_window.py  # 主窗口（4 Tab + 下载/加载器/版本/账号流程）
│       ├── widgets.py      # DownloadProgressDialog + LoaderInstallDialog
│       ├── login_dialog.py
│       └── version_dialogs.py  # 新建/编辑版本对话框 — 新增
├── data/                   # client_config.json / installed.json / .reporter_id（gitignored）
├── requirements.txt
└── .github/workflows/build-release.yml
```

### 4.3 入口与配置

`app/main.py`（[查看](file:///workspace/gpm-client/app/main.py)）：
1. `import gpm_common` 触发适配器注册
2. `ClientConfig.load()`
3. 若无 token/username → 弹 `LoginDialog`（默认 `http://127.0.0.1:8001`），成功后写入 config 并 save
4. **融合体兜底**：`admin_url` 留空时用 `server_url` 兜底，用户填一个地址即可同步 + 上报
5. `MainWindow(config).show()` + `start_reporter(config)`

> MSA 正版登录是**独立的、可选的叠加层**，在主窗口「账号」菜单里触发，不替代 GPM 服务端账号登录。

`ClientConfig` 字段（[查看](file:///workspace/gpm-client/app/config.py)）：
`server_url` / `install_base_dir` / `java_path` / `jvm_args`（默认 `["-Xmx4G","-Xms1G"]`）/ `last_sync_at` / `admin_url` / `reporter_interval` / `client_name` / `reporter_id`（持久化）/ `username` / `token` / `msa_credentials`（dict，存 MSA 登录凭据）

环境变量：`GPM_ADMIN_URL` / `GPM_REPORTER_INTERVAL` / `GPM_CLIENT_NAME`

### 4.4 核心模块

#### sync_manager.py（[查看](file:///workspace/gpm-client/app/sync_manager.py)）
- `ItemStatus.state` ∈ `{not_installed, installed, update_available}`
- 比对 `installed.json` 中的 hash 与服务端 `file_hash`

#### downloader.py（[查看](file:///workspace/gpm-client/app/downloader.py)）
- 多线程分块（支持 Range 且 ≥1MB 走多线程，默认 8 线程）
- 流式 sha256 校验，不匹配删临时文件并抛错
- `os.replace` 原子落盘
- `cancel_event` 每 256KB / 每块检查

#### installer.py（[查看](file:///workspace/gpm-client/app/installer.py)）
- `install_modpack`：CurseForge 风格只解 `overrides/`，否则全解
- `install_mod`：通过 `install_dir_hint` 定位 mods 目录，`shutil.copy2`；新增 `target_dir` 参数支持「另存为」模式

#### launcher.py（[查看](file:///workspace/gpm-client/app/launcher.py)）— 重大升级

`launch()` 签名新增：
```python
def launch(game, install_dir, modpack_meta,
           java_path=None, jvm_args=None, extra_args=None,
           account: Optional[dict] = None,        # 新增：正版账号信息
           game_dir: Optional[str] = None,        # 新增：版本隔离 game_dir
) -> subprocess.Popen:
```
- **账号透传**：`account` 三字段齐全（`username`/`uuid`/`mc_access_token`）才视为正版启动，写入 `LaunchConfig.username/uuid/access_token/user_type="msa"`；否则离线模式
- **版本隔离**：`cwd = config.game_dir or install_dir`——隔离模式下子进程工作目录是 `game_dir`（saves/mods/config 落点），但 classpath/libraries/assets 仍从 `install_dir`（共享根）解析
- **自动内存分配**：`detect_system_memory_mb()`（Windows 用 `GlobalMemoryStatusEx`，Linux 读 `/proc/meminfo`）→ `auto_memory_args()`（总内存 60% 给 MC，封顶 12G，下限 2G，`-Xms = -Xmx`）
- **JVM 优化 flag**：`build_jvm_args()` 在用户未指定时追加 G1GC / ParallelRefProcEnabled / MaxGCPauseMillis=50 / G1NewSizePercent=20 / G1ReservePercent=20 / MaxDirectMemorySize=2G / AlwaysPreTouch / sun.java2d.noddraw 等，逐项 `_flag_key` 去重

#### loader_installer.py（[查看](file:///workspace/gpm-client/app/loader_installer.py)）— 重点

**统一入口** `install_loader(loader, install_dir, mc_version, loader_version, install_base_dir, java_path, progress, cancel_event)`：
- `SUPPORTED_LOADERS = ("fabric", "forge", "neoforge", "quilt")`（vanilla 无需安装器）
- `_INSTALLERS` 字典分发到 `_install_fabric / _install_quilt / _install_forge / _install_neoforge`
- `ProgressCb = Callable[[str, str, int], None]`，stage ∈ `{download, install, done, error}`

**通用辅助**：
- `_cache_dir(install_base_dir)` → `{install_base_dir}/.cache/loaders`（**安装器缓存机制**，已缓存且非空直接复用）
- `_download_to_cache(url, cache_path, ...)`：流式下载到 `.part`，`os.replace` 原子替换
- `_run_java_installer(cmd, ...)`：Popen 合并 stdout/stderr，**按行数推进度 `pct = min(95, lines*5)`**，封顶 95%

**各加载器**：
| Loader | 安装器来源 | 版本解析 | 安装命令 |
|--------|-----------|---------|---------|
| fabric | 固定 `fabric-installer-1.1.0.jar` | `meta.fabricmc.net` 优先 stable | `java -jar fabric-installer.jar client -dir <dir> -mcversion <mc> -loader <loader> -noprofile` |
| quilt | Maven `quilt-installer-{ver}.jar` | `meta.quiltmc.org` 优先 stable | `java -jar quilt-installer.jar install client <mc> --install-dir=<dir> --loader=<loader> --no-profile` |
| forge | `forge-{mc}-{forge}-installer.jar` | `promotions_slim.json` 优先 recommended | `java -jar forge-...-installer.jar --installClient <dir>` |
| neoforge | `neoforge-{ver}-installer.jar` | NeoForge Maven metadata，按 `_parse_version_key` 降序 | `java -jar neoforge-...-installer.jar --installClient <dir>` |

**NeoForge 版本前缀特例**（`_neoforge_version_prefix`）：MC `1.20.1` → `"47"`（继承 Forge 47）；其余去开头 `1.`（`1.20.2`→`20.2`、`1.21`→`21`）。

**NeoForge 版本排序**（`_parse_version_key`）：`beta_flag`（稳定=2 > beta=1）放最前，保证稳定版总排 beta 前。

**与版本隔离集成**：`install_dir` 参数现可传共享游戏根 `game_root(install_base_dir)`，加载器把版本 JSON 写到 `versions/<version_id>/`，libraries 写到共享 `libraries/`。

#### minecraft_installer.py（[查看](file:///workspace/gpm-client/app/minecraft_installer.py)）— 新增

补全原版 MC 运行时（Fabric/Quilt/Forge/NeoForge 安装器只装加载器本身，启动时 Fabric game provider 需在 classpath 找到原版 client jar，否则报 `couldn't locate the game!`）。

- 全部走 **BMCLAPI 镜像**（`bmclapi2.bangbang93.com`），兼容 Mojang 官方 API 路径，仅替换域名
- `ensure_vanilla_version(install_dir, mc_version, ...)` 幂等流程（已存在且 SHA1 校验通过则跳过）：
  1. 版本 JSON → `versions/<mc>/<mc>.json`
  2. client jar → `versions/<mc>/<mc>.jar`（SHA1 校验）
  3. libraries（`_lib_allowed` 平台 rules 过滤，`_lib_path` 转 jar 路径，**16 线程并发下载**）
  4. assets（asset index → objects，每个 hash 落 `assets/objects/<hash[:2]>/<hash>`，16 线程并发）
- `install_dir` 即共享游戏根，所有版本的原版文件跨版本共享

#### java_installer.py（[查看](file:///workspace/gpm-client/app/java_installer.py)）— 新增

按 MC 版本自动安装匹配的 Java 运行时。

- `required_java_major(mc_version)`：MC ≤1.16.5 → Java 8；1.17~1.20.4 → 17；≥1.20.5 → 21；兜底 17
- 下载源（按优先级，全失败抛聚合异常）：
  1. **清华 TUNA 镜像**（`mirrors.tuna.tsinghua.edu.cn/Adoptium/`，解析 autoindex HTML 取最新 zip）
  2. **Adoptium 官方 API**（`api.adoptium.net`，海外回退）
- 安装位置：`<install_base_dir>/.cache/java/jdk-<major>/`；安装包缓存 `<install_base_dir>/.cache/java/jdk-<major>.zip`
- `ensure_java(install_base_dir, mc_version, user_java_path, ...)`：已配置且可运行直接返回；否则按 mc_version 算 Java 大版本 → `find_local_java` 查本地缓存 → 找不到则下载 zip 解压
- `detect_java_major`：运行 `java -version` 解析（Java 8 报 `1.8`，Java 9+ 报大版本号）

#### msa_auth.py（[查看](file:///workspace/gpm-client/app/msa_auth.py)）— 新增

微软账号正版登录，完整 MSA → XBL → XSTS → MC 链路。

- **硬编码**：`CLIENT_ID = "c36a9fb6-4f2a-41ff-90bd-ae7cc92031eb"`（Prism Launcher 公开 client_id，已注册 `http://localhost` loopback 过 Mojang 审核，公开非机密值）、`REDIRECT_PORT = 8917`、`SCOPE = "XboxLive.signin offline_access"`
- 端点：MS `login.microsoftonline.com/consumers`、XBL `user.auth.xboxlive.com`、XSTS `xsts.auth.xboxlive.com`（SandboxId=RETAIL）、MC `api.minecraftservices.com`
- `login_with_browser()`：起本地 HTTP 服务（8917）等回调 → 浏览器登录 → 授权码换 MSA token → XBL（+uhs）→ XSTS（+uhs）→ MC access_token（24h）→ 检查游戏所有权（`mcstore`）→ 取玩家档案（name+id）
- `MsaCredentials`：`username` / `uuid`（无连字符 32 位 hex）/ `mc_access_token` / `ms_refresh_token`（长期，**一次性**）/ `mc_expires_at`；序列化到 `ClientConfig.msa_credentials` 持久化
- `relogin_with_refresh_token()`：用 refresh_token 静默续登（不弹浏览器），重跑整条链路；**refresh_token 一次性，每次刷新换发新的，必须存最新**（`ensure_valid_credentials` 续登后由 `main_window._resolve_launch_account` 立即写回 config 并 save）
- `is_mc_token_valid`：留 60s 缓冲

#### version_manager.py（[查看](file:///workspace/gpm-client/app/version_manager.py)）— 新增

HMCL 风格多版本隔离。

**目录布局**（共享根 `<install_base_dir>/minecraft/`）：
```
minecraft/
  versions/
    1.20.1/                       ← 原版版本（Mojang 标准）
      1.20.1.json                 ← Mojang 版本 JSON（决定 classpath/mainClass/args）
      1.20.1.jar                  ← client jar
      gpm_instance.json           ← GPM 每版本独立配置
    fabric-loader-0.15.7-1.20.1/
      fabric-loader-0.15.7-1.20.1.json
      gpm_instance.json
  libraries/                      ← 跨版本共享
  assets/                         ← 跨版本共享
```

- `version_id` 严格遵循 Mojang 标准版本目录名：vanilla=`1.20.1`、fabric=`fabric-loader-<lv>-<mc>`、quilt=`quilt-loader-<lv>-<mc>`、forge=`<mc>-forge-<fv>`、neoforge=`neoforge-<ver>`
- `VersionInstance` dataclass：`version_id` / `display_name` / `game_version` / `mod_loader` / `mod_loader_version` / `java_path`（空=继承全局）/ `jvm_args`（空=继承全局）/ `isolated`（True=存档/模组/配置隔离到 `versions/<id>/`，False=共享根）/ `last_played` / `created_at` / `ready`（client jar 是否就绪）/ `version_dir`
- **与 installed.json 关系**：两套独立追踪系统，互不耦合。`installed.json` 记录从 GPM 服务端下载的 modpacks/mods；`gpm_instance.json` 记录每版本独立启动配置。version_manager **完全不读写 installed.json**
- 关键函数：`game_root()` / `version_dir()` / `list_versions()`（扫描 `versions/<id>/<id>.json`）/ `create_version()`（下载原版 + 装加载器 + 写 instance 配置）/ `delete_version()`（严格安全检查，仅删 `versions/<id>/` 本身，不影响共享 libraries/assets）/ `resolve_game_dir()`（`isolated=True` → `inst.version_dir`；`False` → root）/ `touch_last_played()` / `fetch_mc_version_ids()`（BMCLAPI 拉版本清单）
- BMCLAPI manifest：`https://bmclapi2.bangbang93.com/mc/game/version_manifest_v2.json`

#### reporter.py（[查看](file:///workspace/gpm-client/app/reporter.py)）
- `_build_heartbeat`：`kind="client"`、`username`（后台按用户去重）、`light=GREEN`（在线即绿）
- `metrics`：`installed_modpacks`（仅 modpacks 条目）/ `installed_modpack_count` / `last_sync_at` / `platform` / `install_base_dir`

### 4.5 UI

#### main_window.py（[查看](file:/workspace/gpm-client/app/ui/main_window.py)）

**跨线程信号机制**（注释强调）：工作线程通过 `_sig_*` 信号投递到主线程（AutoConnection 跨线程自动 QueuedConnection，`QDialog.exec()` 嵌套事件循环能正确接收）：
- 下载：`_sig_progress(int,int)` / `_sig_status(str)` / `_sig_close_dialog(int)` / `_sig_statusbar(str,int)` / `_sig_fail(str,str)`
- 加载器：`_sig_loader_progress(str,str,int)` / `_sig_loader_done(str)` / `_sig_loader_failed(str)`
- 版本清单异步回填：`_sig_ver_versions(list)`

**4 Tab**：整合包页（列表 + 详情 + 下载/启动）/ 模组页（表格 + 内嵌下载按钮）/ 设置页 / **版本管理页（新增）**。

**菜单栏「账号(A)」**（新增）：微软账号登录…（`_on_msa_login` 后台线程跑 `login_with_browser()`）、退出微软账号、当前状态显示（离线模式 / `用户名（正版）`）。

**下载流程** `_start_download`：
1. 创建 `DownloadProgressDialog`，`canceled` 连 `_cancel_event.set()`
2. 启动 `_download_worker` daemon 线程，`exec()` 阻塞
3. **整合包下载成功后**：`if kind == "modpacks" and accepted: self._maybe_install_loader(item)`

**加载器安装流程** `_maybe_install_loader`（增强：所有加载器含 vanilla 启动前都先确保 Java + 原版文件）：
- `_ensure_java(mc_version)` → `_ensure_vanilla_version(item)` → `loader == "vanilla"` 直接 return → 无 `game_version` 提示 → `loader not in SUPPORTED_LOADERS` 提示手动安装 → 创建 `LoaderInstallDialog` 启动 `_loader_worker`

**版本管理 Tab**（`_build_versions_tab`）：工具栏 新建版本 / 启动 / 配置 / 打开目录 / 删除 / 刷新；列表项显示 `显示名 [loader ver / MC ver] · 就绪/未完成 · 隔离/共享 · 上次启动时间`。

**版本启动流程**（`_ver_on_launch`）：
1. `_ensure_java(mc_version)`
2. `_ver_ensure_files(inst)`（未就绪则下载原版文件）
3. `_resolve_launch_account`（MSA token 有效直接用；过期静默续登，刷新成功立即持久化新 refresh_token；续登失败三选一：重新浏览器登录 / 离线启动 / 取消）
4. `launch(game="minecraft", install_dir=root, modpack_meta={version_id, ...}, account=account, game_dir=vm_resolve_game_dir(root, inst))`
5. `vm_touch_last_played`

#### widgets.py（[查看](file:///workspace/gpm-client/app/ui/widgets.py)）

`LoaderInstallDialog` 多阶段进度对话框（新增 `stages=` 参数支持自定义阶段，Java 下载复用）：
- 默认 `_STAGES = [("download", "下载安装器"), ("install", "安装加载器"), ("done", "完成")]`
- 每阶段一行：状态点 `QLabel` + 阶段名 + `QProgressBar`
- 状态点配色：当前 stage `●` 蓝 `#0A84FF`，已过 `●` 绿 `#30D158`，未到 `○` 灰 `#48484A`
- 日志区深色背景，保留最后 8 行
- `on_done`：所有阶段绿 100%，禁用取消，启用完成按钮
- `on_failed`：detail 红 `#FF453A`，取消按钮改"关闭"

#### version_dialogs.py（[查看](file:/workspace/gpm-client/app/ui/version_dialogs.py)）— 新增

- `CreateVersionDialog`：显示名称 / MC 版本（QComboBox 可编辑，打开时后台异步拉 BMCLAPI 清单通过 `_sig_ver_versions` 回填，失败回退可编辑输入）/ 模组加载器 / 加载器版本 / Java 路径 / 版本隔离复选框（默认勾选）
- `EditVersionDialog`：编辑选中版本的 `display_name` / `java_path` / `jvm_args` / `isolated`，写回 `gpm_instance.json`

#### login_dialog.py
`LoginDialog` 返回 `(server_url, username, token)`，回车链（用户名→密码→登录）。

### 4.6 加载器自动安装完整流程（重点）

```
用户点击"下载"整合包
   ↓
_start_download → _download_worker 线程
   ↓ download_file（多线程分块 + sha256 校验）
   ↓ install_modpack（解压，CurseForge 只解 overrides/）
   ↓ 记录 installed.json
   ↓ emit _sig_close_dialog(0)
exec() 返回 Accepted
   ↓
_maybe_install_loader(item)
   ↓ _ensure_java(mc_version)            ← 新增：所有加载器都先确保 Java
   ↓ _ensure_vanilla_version(item)       ← 新增：下载原版文件
   ↓ loader == "vanilla"? → return
   ↓ 创建 LoaderInstallDialog
   ↓ 启动 _loader_worker 线程
   ↓ install_loader(loader, install_dir, mc_version, loader_version, ...)
       ↓ _download_to_cache（安装器缓存到 .cache/loaders）
       ↓ _resolve_xxx_version（查 Meta API / Maven）
       ↓ _run_java_installer（java -jar ... 安装到 install_dir）
       ↓ progress("done", ..., 100)
   ↓ emit _sig_loader_done / _sig_loader_failed
exec() 返回
   ↓
用户点击"启动游戏"
   ↓ _on_launch → _ensure_java + _ensure_vanilla_version + _resolve_launch_account
   ↓ GameAdapterRegistry.require("minecraft")
   ↓ adapter.build_launch_command（解析版本 JSON + classpath + 账号占位符 + game_dir）
   ↓ subprocess.Popen(cmd, cwd=game_dir or install_dir)
```

### 4.7 构建发布

`.github/workflows/build-release.yml`：Nuitka 单文件 exe
- 触发：`workflow_dispatch` + `push tags: ['v*']`
- `windows-latest`，Python 3.11
- `--onefile --standalone --windows-console-mode=disable --enable-plugin=pyside6`
- `--include-package=gpm_common --include-package=app`
- 生成 `gpm-client.exe`，配置存 exe 同级 `data/`

---

## 五、跨仓库协作流程

### 5.1 整合包从上传到启动的完整链路

```
1. 管理员在 gpm-web-admin/admin 上传整合包
   → POST /api/v1/modpacks（mod_loader=auto）
   → adapter.detect_metadata 自动识别 game_version/mod_loader（含 neoforge）
   → storage.save_modpack（校验 + 存储 + meta.json）

2. 客户端 gpm-client 启动 → GET /api/v1/sync 拉取 enabled 条目
   → 比对 installed.json 标记状态

3. 用户点"下载"
   → download_file（多线程 + sha256 校验）
   → install_modpack（解压）
   → _maybe_install_loader（自动安装加载器）
       → _ensure_java（缺 Java 则按 MC 版本下载 Java 8/17/21）
       → _ensure_vanilla_version（下载原版 MC 文件到共享根）
       → 下载加载器安装器到 .cache/loaders
       → 解析最新 loader 版本
       → java -jar 安装到 install_dir

4. 用户点"启动"
   → _resolve_launch_account（MSA token 有效直接用；过期静默续登；失败询问）
   → adapter.build_launch_command（解析版本 JSON + classpath + 账号占位符 + game_dir）
   → subprocess.Popen 启动游戏
```

### 5.2 心跳上报与仪表盘

```
各端启动（admin_url 非空）
   → Reporter 后台线程（默认 10s 间隔，启动立即上报一次）
   → POST /api/v1/report（Heartbeat：状态 + modpacks/mods 列表 + light + admin_users）
   → web-admin ReportStore.record（按 reporter_id/username 去重）
   → sync_admin_users（合并管理员账号）

前端 index.html 每 5s GET /api/v1/dashboard（需 admin）
   → ReportStore.aggregate
   → 总体灯色 + 按 kind 分组 + 推送条目聚合
```

### 5.3 用户同步机制

- 服务端上报时 `metrics.admin_users` 携带管理员 hash 列表
- web-admin `sync_admin_users(admin_users, source=reporter_id)`：
  - 先清除该 source 之前同步的带 `_source` 用户
  - 本地用户（无 `_source`）hash 不同则更新
  - 远端用户写入 `{hash, role:"admin", _source:source}`
- 实现"一处建管理员，多处可登录"

### 5.4 mod_loader 自动识别链路（跨 2 仓库）

```
gpm-client 上传表单（mod_loader=auto）
   → gpm-web-admin 路由调 adapter.detect_metadata
   → gpm-common MinecraftAdapter 识别（CurseForge/Modrinth/裸包）
   → 返回 mod_loader（含 neoforge）
   → 写入 meta.json
```

### 5.5 正版账号启动链路（跨 2 仓库）

```
gpm-client「账号」菜单 → msa_auth.login_with_browser（MSA→XBL→XSTS→MC）
   → MsaCredentials 存入 ClientConfig.msa_credentials
   → 启动时 _resolve_launch_account
       → token 有效直接用
       → 过期则 relogin_with_refresh_token 续登（refresh_token 一次性，存最新）
   → launch(account={username, uuid, mc_access_token})
   → gpm-common LaunchConfig(username/uuid/access_token/user_type="msa")
   → adapter.build_launch_command 展开正版账号占位符
```

---

## 六、开发约定

### 6.1 协议与模型
- 所有端共用 `gpm_common.models`，避免字段不一致
- `API_PREFIX="/api/v1"`、`API_VERSION="1.0.0"`，所有端必须一致
- 路由路径用 `gpm_common.route("/xxx")` 拼接

### 6.2 路径安全
- 文件操作一律走 `gpm_common.safe_join` 防穿越
- id / 文件名经 `_sanitize` / `_sanitize_filename` 清理

### 6.3 认证模型
- JWT（HS256），默认 24h 过期，secret 来自 `GPM_AUTH_SECRET`（未设则进程内随机）
- 读操作（sync/list/download/status/games/config 读）开放
- 写操作（上传/删除/PATCH/改密/用户管理/config 改/仪表盘/系统更新）需 `require_admin`（admin role）
- 登录返回 token + role，前端按 role 控制可见性

### 6.4 线程通信（客户端）
- UI 用 Qt Signal/Slot + QueuedConnection
- 工作线程（`threading.Thread`）通过 `_sig_*` 信号投递到主线程
- `QDialog.exec()` 嵌套事件循环能正确接收 queued 事件
- 取消用 `threading.Event` 贯穿下载器与加载器安装器

### 6.5 存储
- 无数据库，基于文件系统 JSON 元数据
- 整合包/模组：`data/{kind}/{id}/{meta.json, <file>}`
- 用户：`data/users.json`；配置：`data/server.json`；上报 ID：`data/.reporter_id`
- 版本隔离配置：`<install_base>/minecraft/versions/<id>/gpm_instance.json`（仅客户端）

### 6.6 错误处理
- 业务异常用 `GamePushError(code, status_code, message, details)`
- 认证异常用 `AuthError(message, status_code)`
- 返回体统一 `{"error":{"code","message","details"}}`

---

## 七、扩展指南

### 7.1 新增游戏

只需在 `gpm-common` 实现 + 注册，**其他仓库无需改动**：

```python
# gpm-common/gpm_common/adapters/my_game.py
from gpm_common.game_adapter import GameAdapter, GameAdapterRegistry
from gpm_common.models import GameInfo, LaunchConfig

class MyGameAdapter(GameAdapter):
    game_name = "my_game"
    display_name = "My Game"

    def game_info(self) -> GameInfo:
        return GameInfo(name="my_game", display_name="My Game", adapter="MyGameAdapter", enabled=True)

    def validate_modpack(self, archive_path: str) -> bool: ...
    def build_launch_command(self, install_dir, launch_config, modpack_meta) -> list[str]: ...
    # 可选：detect_metadata / install_dir_hint / supported_mod_loaders
```

在 `gpm_common/adapters/__init__.py` 的 `register_builtin_adapters()` 中注册。

### 7.2 新增模组加载器

1. `gpm-common/adapters/minecraft.py`：
   - `supported_mod_loaders()` 加入新 loader
   - `build_launch_command` 加分支 + `_find_xxx_jar`（或版本 JSON 推断）
   - `detect_metadata` / `detect_mod_metadata` 加识别规则
2. `gpm-client/app/loader_installer.py`：
   - 实现 `_install_xxx`
   - 加入 `_INSTALLERS` 字典和 `SUPPORTED_LOADERS`
3. 前端下拉框（gpm-web-admin `admin.html` / `admin.js`，客户端 GUI `ModpackMetaDialog`）加入选项

### 7.3 新增 API

1. `gpm-common/protocol.py` 加路由常量（可选）
2. `gpm-web-admin/app/routes/xxx.py` 实现，`router = APIRouter()`
3. `gpm-web-admin/app/main.py` 注册 `app.include_router(xxx.router)`
4. 鉴权按需加 `Depends(require_token(settings.auth_secret))`（任何登录用户）或 `Depends(require_admin(settings.auth_secret))`（仅 admin）

---

## 八、已知问题与落差

### 8.1 gpm-web-admin
- 写操作挂了 `require_admin` 但未额外校验"是否为 admin"之外的细粒度权限，任何 admin 用户均可增删用户（靠 UI 行为约束）
- 自动更新依赖 systemd `Restart=always`，非 systemd 环境需手动重启

### 8.2 gpm-client
- 首次运行 Nuitka exe 稍慢（自解压临时文件）
- `admin_url` 为空时用 `server_url` 兜底（融合体设计），独立后台场景需手动配置
- MSA `refresh_token` 是一次性的，每次续登换发新的，必须立即持久化（已在 `_resolve_launch_account` 处理，但若续登后未 save config 会失效）

### 8.3 跨仓库
- `gpm-common` 模型 `mod_loader` 字段注释仅写 4 项，实际支持 5 项（含 neoforge）—— 文档与代码不一致
- 各仓库 `gpm-common` 依赖从 GitHub `yzgmc/gpm-common` 安装，需保持版本同步

---

## 九、给后续 AI 的开发指引

### 9.1 快速上手

1. **先读本文件**了解全局架构
2. 改动前先读对应文件（用 file:/// 链接定位）
3. 3 个仓库的依赖关系：`gpm-common` 是基础，改它会影响所有端
4. 提交规范：各子仓库独立提交，主仓库（本仓库）用于存放功能说明文档

### 9.2 仓库地址

| 仓库 | GitHub |
|------|--------|
| 主仓库（本文档所在） | `https://github.com/yzgmc/Game-Push-Manager` |
| gpm-common | `https://github.com/yzgmc/gpm-common` |
| gpm-web-admin | `https://github.com/yzgmc/gpm-web-admin` |
| gpm-client | `https://github.com/yzgmc/gpm-client` |

### 9.3 本地开发环境

```bash
# 1. 克隆 3 个仓库到同一父目录
git clone https://github.com/yzgmc/gpm-common
git clone https://github.com/yzgmc/gpm-web-admin
git clone https://github.com/yzgmc/gpm-client

# 2. 安装共享库（开发模式）
cd gpm-common && pip install -e .

# 3. 各端安装依赖
cd ../gpm-web-admin && pip install -r requirements.txt
cd ../gpm-client && pip install -r requirements.txt

# 4. 运行（融合体最常用，单进程即可调试全部流程）
cd gpm-web-admin && python run.py    # http://localhost:8001
# 或客户端
cd gpm-client && python run.py       # 需先启动融合体
```

### 9.4 调试技巧

- **融合体自上报**：启动 gpm-web-admin 后，仪表盘会自动出现一个 `web-server` 上报端（自己上报给自己）
- **默认账号**：`admin / admin123`
- **数据目录**：源码运行在仓库根 `data/`，打包后在 exe 同级 `data/`
- **日志**：`gpm-deploy.sh` 部署的用 `journalctl -u gpm-web-admin -f`
- **协议版本不一致**会报 `PROTOCOL_MISMATCH`，确保各端 `gpm-common` 版本同步

### 9.5 常见任务速查

| 任务 | 改哪里 |
|------|--------|
| 加新游戏 | `gpm-common/gpm_common/adapters/` 新建 + 注册 |
| 加新加载器 | `gpm-common/adapters/minecraft.py` + `gpm-client/loader_installer.py` + 前端下拉 |
| 加新 API | `gpm-web-admin/app/routes/` + `app/main.py` 注册 |
| 改数据模型 | `gpm-common/gpm_common/models.py`（注意向后兼容） |
| 改灯色阈值 | 环境变量 `GPM_LIGHT_*` 或 `gpm-common/lights.py` |
| 改认证 | `gpm-common/auth.py` |
| 改仪表盘聚合 | `gpm-web-admin/app/report_store.py` 的 `aggregate()` |
| 改加载器安装 | `gpm-client/app/loader_installer.py` |
| 改自动更新 | `gpm-web-admin/app/updater.py` |
| 改正版登录 | `gpm-client/app/msa_auth.py` |
| 改版本隔离 | `gpm-client/app/version_manager.py` |
| 改启动命令生成 | `gpm-common/gpm_common/adapters/minecraft.py` 的 `build_launch_command` |
| 改 Java 自动安装 | `gpm-client/app/java_installer.py` |
| 改原版 MC 下载 | `gpm-client/app/minecraft_installer.py` |

### 9.6 提交与发布

- 各子仓库独立 git 提交，建议 conventional commit
- 打 tag 触发 GitHub Actions 自动构建 exe（`v*` tag，仅 gpm-client 有构建工作流）
- `gpm-common` 改动需推送后，其他仓库 `pip install git+https://github.com/yzgmc/gpm-common.git` 更新
- 主仓库（本仓库）用于存放本功能说明文档，更新文档直接提交 main 分支

---

## 附：关键文件快速索引

### gpm-common
- [models.py](file:///workspace/gpm-common/gpm_common/models.py)
- [protocol.py](file:///workspace/gpm-common/gpm_common/protocol.py)
- [game_adapter.py](file:///workspace/gpm-common/gpm_common/game_adapter.py)
- [adapters/minecraft.py](file:///workspace/gpm-common/gpm_common/adapters/minecraft.py)
- [heartbeat.py](file:///workspace/gpm-common/gpm_common/heartbeat.py)
- [reporter.py](file:///workspace/gpm-common/gpm_common/reporter.py)
- [auth.py](file:///workspace/gpm-common/gpm_common/auth.py)
- [lights.py](file:///workspace/gpm-common/gpm_common/lights.py)
- [storage.py](file:///workspace/gpm-common/gpm_common/storage.py)
- [gpm-deploy.sh](file:///workspace/gpm-common/gpm-deploy.sh)

### gpm-web-admin
- [app/main.py](file:///workspace/gpm-web-admin/app/main.py)
- [app/config.py](file:///workspace/gpm-web-admin/app/config.py)
- [app/report_store.py](file:///workspace/gpm-web-admin/app/report_store.py)
- [app/updater.py](file:///workspace/gpm-web-admin/app/updater.py)
- [app/reporter.py](file:///workspace/gpm-web-admin/app/reporter.py)
- [app/routes/report.py](file:///workspace/gpm-web-admin/app/routes/report.py)
- [app/routes/dashboard.py](file:///workspace/gpm-web-admin/app/routes/dashboard.py)
- [app/routes/update.py](file:///workspace/gpm-web-admin/app/routes/update.py)
- [app/routes/auth.py](file:///workspace/gpm-web-admin/app/routes/auth.py)
- [static/admin.html](file:///workspace/gpm-web-admin/static/admin.html)
- [static/admin.js](file:///workspace/gpm-web-admin/static/admin.js)
- [static/app.js](file:///workspace/gpm-web-admin/static/app.js)

### gpm-client
- [app/main.py](file:///workspace/gpm-client/app/main.py)
- [app/config.py](file:///workspace/gpm-client/app/config.py)
- [app/loader_installer.py](file:///workspace/gpm-client/app/loader_installer.py)
- [app/minecraft_installer.py](file:///workspace/gpm-client/app/minecraft_installer.py)
- [app/java_installer.py](file:///workspace/gpm-client/app/java_installer.py)
- [app/msa_auth.py](file:///workspace/gpm-client/app/msa_auth.py)
- [app/version_manager.py](file:///workspace/gpm-client/app/version_manager.py)
- [app/downloader.py](file:///workspace/gpm-client/app/downloader.py)
- [app/installer.py](file:///workspace/gpm-client/app/installer.py)
- [app/launcher.py](file:///workspace/gpm-client/app/launcher.py)
- [app/reporter.py](file:///workspace/gpm-client/app/reporter.py)
- [app/ui/main_window.py](file:///workspace/gpm-client/app/ui/main_window.py)
- [app/ui/widgets.py](file:///workspace/gpm-client/app/ui/widgets.py)
- [app/ui/version_dialogs.py](file:///workspace/gpm-client/app/ui/version_dialogs.py)
- [app/ui/login_dialog.py](file:///workspace/gpm-client/app/ui/login_dialog.py)

---

*本文档由深度研究 3 个仓库源码后编写，覆盖全部模块、路由、关键函数与协作流程。后续 AI 仅需阅读本文件即可继续开发，无需重新探索。*
