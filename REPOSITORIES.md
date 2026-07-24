# Game Push Manager — 仓库功能总览

> 本文档面向**后续接手的 AI / 开发者**：仅阅读本文件即可掌握 4 个子仓库的全部功能、架构、协作流程与扩展方式，无需重新探索代码。
> 文档基于实际源码逐文件核对编写，最后更新与代码同步。
> 主仓库（Game-Push-Manager）远程：`https://github.com/yzgmc/Game-Push-Manager`

---

## 目录

- [一、项目总览](#一项目总览)
- [二、gpm-common（共享库）](#二gpm-common共享库)
- [三、gpm-server（Windows 服务端）](#三gpm-serverwindows-服务端)
- [四、gpm-web-admin（融合体：网页后台 + 服务端）](#四gpm-web-admin融合体网页后台--服务端)
- [五、gpm-client（Windows 客户端）](#五gpm-clientwindows-客户端)
- [六、跨仓库协作流程](#六跨仓库协作流程)
- [七、开发约定](#七开发约定)
- [八、扩展指南](#八扩展指南)
- [九、已知问题与落差](#九已知问题与落差)
- [十、给后续 AI 的开发指引](#十给后续-ai-的开发指引)

---

## 一、项目总览

### 1.1 项目定位

Game Push Manager（GPM）是一套**游戏整合包 / 模组推送分发与监控管理系统**，核心场景：

- 管理员通过网页后台上传整合包 / 模组，管理用户、查看各端运行状态。
- 服务端对外提供同步 / 下载 API。
- Windows 客户端从服务端拉取条目，下载整合包后**自动安装模组加载器**并一键启动游戏。
- 所有端通过 **Push 模型**主动向后台上报心跳，后台聚合展示仪表盘 + 状态指示灯。

当前原生支持 **Minecraft**，通过 `gpm-common` 的游戏适配器机制可扩展其他游戏。

### 1.2 四个仓库关系

```
                        ┌─────────────────────────────┐
                        │      gpm-common（共享库）      │
                        │  数据模型 / 协议 / 适配器 /     │
                        │  心跳 / 认证 / 灯色 / 存储      │
                        └──────────────┬──────────────┘
                                       │ pip install -e .
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
   ┌──────────▼──────────┐  ┌──────────▼──────────┐  ┌──────────▼──────────┐
   │   gpm-server        │  │  gpm-web-admin      │  │   gpm-client        │
   │ （Windows 服务端）    │  │ （融合体：后台+服务端）│  │ （Windows 客户端）    │
   │ FastAPI + PySide6   │  │ FastAPI + 静态前端   │  │ PySide6 桌面应用     │
   └──────────┬──────────┘  └──────────┬──────────┘  └──────────┬──────────┘
              │                        │                        │
              └────────────┬───────────┴────────────────────────┘
                           │ POST /api/v1/report（Heartbeat）
                           ▼
                  gpm-web-admin 内存 ReportStore
                  → 仪表盘聚合 + 灯色
```

**关键设计：融合体**

`gpm-web-admin` 是"网页后台 + 网页服务端"的**单进程融合体**：同时承担
1. 接收各端心跳的后台角色（`/api/v1/report`、`/api/v1/dashboard`）；
2. 提供上传 / 同步 / 下载 / 用户管理 / 自动更新的服务端角色。

启动后默认把 `admin_url` 指向自己，**自己上报给自己**，自动纳入仪表盘。`gpm-server` 则是面向 Windows 服务器环境的独立服务端（功能与融合体的服务端部分等价）。

### 1.3 技术栈

| 仓库 | 语言 | 框架 | GUI | 打包 |
|------|------|------|-----|------|
| gpm-common | Python ≥3.9 | Pydantic v2 + httpx（可选 fastapi） | — | pip 包 |
| gpm-server | Python ≥3.9 | FastAPI + uvicorn | PySide6 | Nuitka 单文件 exe |
| gpm-web-admin | Python ≥3.9 | FastAPI + uvicorn | 纯静态 HTML/JS | Nuitka / systemd |
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
| `LaunchConfig` | `java_path` / `jvm_args` / `extra_args` |
| `ModpackBase` | `name` / `version` / `game` / `game_version` / `mod_loader`（默认 `vanilla`）/ `mod_loader_version` / `description` / `enabled` |
| `ModpackCreate` | 继承 ModpackBase（上传用） |
| `Modpack` | 继承 ModpackBase + `id` / `file_name` / `file_size` / `file_hash` / `created_at` / `updated_at` |
| `ModBase` | `name` / `version` / `game` / `game_version` / `mod_loader` / `mod_loader_version` / `modpack_id` / `description` / `enabled` |
| `ModCreate` / `Mod` | 同 Modpack 模式 |
| `SyncResponse` | `protocol_version` / `server_name` / `modpacks` / `mods` / `games` / `server_time` |
| `StatusResponse` | `server_name` / `server_kind` / `status` / `protocol_version` / `uptime_seconds` / `modpack_count` / `mod_count` / `storage_used_bytes` / `started_at` |

> ⚠️ `mod_loader` 字段注释仅写 `vanilla/forge/fabric/quilt`，但 `MinecraftAdapter.supported_mod_loaders()` 实际返回 5 项（含 `neoforge`）。模型本身不限制取值。

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
- `build_launch_command(install_dir, launch_config, modpack_meta)`：
  - `[java] + jvm_args` 起头（默认 `["-Xmx4G", "-Xms1G"]`）
  - **vanilla**：`-jar <install_dir>/minecraft_server.<game_version>.jar --gameDir <install_dir>`
  - **forge**：`-jar <forge_jar> --gameDir <install_dir>`（`_find_forge_jar` 定位）
  - **neoforge**：`-jar <neo_jar> --gameDir <install_dir>`（`_find_neoforge_jar` 定位）
  - **fabric / quilt**：`-jar <loader_jar> --gameDir <install_dir> --gameVersion <game_version>`（`_find_fabric_loader_jar` 定位）
  - 末尾 `extend(extra_args)`
- jar 定位辅助：`_find_forge_jar` / `_find_neoforge_jar` / `_find_fabric_loader_jar`（遍历目录匹配关键字，优先匹配 loader_version）
- `_detect_java()`：依次 `which(java.exe / javaw.exe / java)`，兜底 `"java"`

#### heartbeat.py（[查看](file:///workspace/gpm-common/gpm_common/heartbeat.py)）

`Heartbeat(BaseModel)`：
- `reporter_id` / `kind`（`windows-server|web-server|client`）/ `name` / `username?`（客户端登录后携带，后台按用户去重）/ `base_url?` / `status` / `protocol_version` / `sent_at` / `light?: Light` / `metrics: dict` / `extra: dict`

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
- `require_token(secret)`：返回 FastAPI 依赖（**局部导入 fastapi.Header**）
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

## 三、gpm-server（Windows 服务端）

### 3.1 定位

面向 Windows 服务器环境的独立服务端，与 `gpm-web-admin` 的服务端部分**共用同一套 `gpm-common` API 契约**，仅部署位置不同。可注册为 Windows 服务长期运行。

- GitHub：`https://github.com/yzgmc/gpm-server`

### 3.2 目录结构

```
gpm-server/
├── run.py                   # 命令行启动 uvicorn
├── run_gui.py               # PySide6 GUI 启动（含淡入动画）
├── app/
│   ├── main.py              # FastAPI app 工厂 + 路由注册 + 静态挂载 + 生命周期
│   ├── config.py            # Settings（环境变量 + users.json + server.json）
│   ├── storage.py           # 整合包/模组文件存储 + meta.json
│   ├── server_info.py       # 运行时统计单例
│   ├── reporter.py          # 心跳上报线程
│   ├── gui.py               # PySide6 GUI（ServerWindow 等）
│   └── routes/
│       ├── auth.py          # 登录/改密/用户管理
│       ├── config.py        # 运行时配置
│       ├── games.py         # 游戏列表
│       ├── status.py        # 状态
│       ├── sync.py          # 客户端同步
│       ├── modpacks.py      # 整合包 CRUD（含自动识别）
│       └── mods.py          # 模组 CRUD
├── static/                  # admin.html / admin.js / login.html / index.html / css
├── requirements.txt
└── .github/workflows/build-release.yml
```

### 3.3 入口与配置

- `run.py`：`uvicorn.run("app.main:app", host, port)`
- `run_gui.py`：`ServerThread` 后台跑 uvicorn + `ServerWindow` GUI（窗口淡入动画 300ms）
- `app/main.py`（[查看](file:///workspace/gpm-server/app/main.py)）：
  - `import gpm_common` 触发适配器注册
  - CORS `allow_origins=["*"]`
  - 中间件 `_count_requests` 计请求；异常处理 `GamePushError` / `AuthError`
  - 路由注册顺序：auth → config → games → status → sync → modpacks → mods
  - 静态挂载 `/static`；页面路由 `GET /` → admin.html、`/login`、`/admin`、`/api/info`
  - startup 启动 reporter，shutdown 停止

**Settings 环境变量**（[查看](file:///workspace/gpm-server/app/config.py)）：

| 变量 | 默认 | 说明 |
|------|------|------|
| `GPM_HOST` | `0.0.0.0` | 监听地址 |
| `GPM_PORT` | `8000` | 监听端口 |
| `GPM_DATA_DIR` | exe 同级 / 仓库根 `data/` | 数据目录 |
| `GPM_SERVER_NAME` | `gpm-windows-server` | 服务名 |
| `GPM_MAX_UPLOAD_MB` | `4096` | 上传上限 |
| `GPM_AUTH_SECRET` | 随机 | JWT secret（未设则重启失效） |
| `GPM_ADMIN_URL` | 空 | 后台地址，非空则启动上报 |
| `GPM_PUBLIC_BASE_URL` | `http://127.0.0.1:<port>` | 上报可访问地址 |
| `GPM_REPORTER_INTERVAL` | `10` | 上报间隔秒 |
| `GPM_REPORTER_ID` | 自动生成 | 上报端 ID |
| `GPM_USERS` | — | `user:hash[:role],...` 只读用户表 |
| `GPM_LIGHT_DISK_YELLOW/RED/ERROR_RED` | 85/95/10 | 灯色阈值 |

### 3.4 路由清单（前缀 `/api/v1`）

| 方法 | 路径 | 鉴权 | 说明 |
|------|------|------|------|
| POST | `/auth/login` | 无 | 登录返回 token（24h） |
| PUT | `/auth/password` | token | 改自己密码 |
| GET | `/users` | token | 用户列表 |
| POST | `/users` | token | 添加用户 |
| PATCH | `/users/{username}` | token | 改角色 |
| DELETE | `/users/{username}` | token | 删用户（至少留一个 admin） |
| GET | `/modpacks` | 无 | 整合包列表 |
| POST | `/modpacks` | token | 上传（自动识别 loader/version） |
| GET | `/modpacks/{id}` | 无 | 详情 |
| PATCH | `/modpacks/{id}` | token | 改元数据/上下架 |
| GET | `/modpacks/{id}/download` | 无 | 下载文件 |
| DELETE | `/modpacks/{id}` | token | 删除 |
| GET/POST/PATCH/DELETE | `/mods/...` | 同上 | 模组 CRUD |
| GET | `/sync` | 无 | 客户端同步（仅 enabled） |
| GET | `/status` | 无 | 服务端状态 |
| GET | `/games` | 无 | 游戏列表 |
| GET | `/config` | 无 | 读运行时配置 |
| PUT | `/config` | token | 改配置 + 热重启 reporter |

### 3.5 存储结构

```
data/
├── users.json              # {username: {hash, role}}
├── server.json             # 运行时配置（admin_url 等）
├── .reporter_id            # 持久化上报 ID
├── modpacks/<id>/{meta.json, <原文件名>}
└── mods/<id>/{meta.json, <原文件名>}
```

### 3.6 mod_loader 自动识别链路（[查看](file:///workspace/gpm-server/app/routes/modpacks.py)）

上传时表单 `mod_loader` 默认 `"auto"`，`game_version` 默认空。触发条件 `not game_version or mod_loader in ("", "auto")`：
1. `adapter = GameAdapterRegistry.get(game)`（`.get()` 不抛错）
2. 调 `adapter.detect_metadata(tmp_path)`（modpack）或 `detect_mod_metadata`（mod）
3. 用识别结果填回 `game_version` / `mod_loader`（**含 neoforge**）/ `mod_loader_version`
4. `except Exception: pass` 容错，识别失败不阻断上传
5. 返回值附 `auto_detected: bool`

> 服务端本身不写识别逻辑，**完全委托给 gpm_common 适配器**。

### 3.7 心跳上报（[查看](file:///workspace/gpm-server/app/reporter.py)）

- 启动条件：`settings.admin_url` 非空
- `_build_heartbeat()`：含 modpacks/mods 完整列表 + `admin_users`（含 hash，供后台同步）+ `light`（`compute_server_light`）
- `PUT /api/v1/config` 后 `restart_reporter()` 热生效

### 3.8 GUI（[查看](file:///workspace/gpm-server/app/gui.py)）

`ServerWindow` 4 个 Tab：整合包管理 / 模组管理 / 用户管理 / 配置。关键类：
- `ApiWorker(QThread)` / `UploadWorker(QThread)`：HTTP 工作线程
- `LoginDialog` / `ModpackMetaDialog`（loader 下拉含 neoforge）/ `ModMetaDialog` / `ChangePasswordDialog` / `AddUserDialog`
- `ServerThread`：后台 uvicorn
- 所有 API 调用打到 `http://127.0.0.1:{port}`

### 3.9 已知落差

- `static/admin.html` 有"系统更新" Tab，`admin.js` 调用 `/api/v1/update/*` 四端点，但 `app/routes/` **无对应实现**（功能未实现）
- `static/index.html` 引用 `/static/app.js` 但该文件不存在，且未被任何路由暴露（遗留页面）

---

## 四、gpm-web-admin（融合体：网页后台 + 服务端）

### 4.1 定位

**单进程融合体**：同时是网页后台（接收心跳 + 仪表盘）+ 网页服务端（上传/同步/管理）。启动后 `admin_url` 默认自指向，自己上报给自己。

- GitHub：`https://github.com/yzgmc/gpm-web-admin`
- 默认端口 `8001`（`gpm-deploy.sh` 部署端口）

### 4.2 目录结构

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
│   ├── admin.js             # 管理前端
│   ├── login.html
│   └── *.css
├── requirements.txt
└── README.md
```

### 4.3 入口与配置

`app/main.py`（[查看](file:///workspace/gpm-web-admin/app/main.py)）路由分两组：
```
# 后台路由
report.router     # POST /api/v1/report（无鉴权）
dashboard.router  # GET  /api/v1/dashboard（需 token）
# 服务端路由
auth / config / games / status / sync / modpacks / mods / update
```
- 页面：`GET /` → index.html（仪表盘）、`/login`、`/admin`
- `GET /api/info`：`service="gpm-web-admin-fusion"`、`kind`、`reporting_to`
- startup：`start_reporter()` + `start_updater()`；shutdown：`stop_reporter()` + `stop_updater()`

Settings 与 gpm-server 基本一致，差异：
- `server_kind = "web-server"`（固定）
- `port` 默认 `8001`
- `admin_url` 默认 `http://127.0.0.1:{port}`（自指向）
- `stale_seconds` 默认 `30`（离线判定阈值）
- 新增 `sync_admin_users(admin_users, source)`：合并其他服务端上报的管理员（带 `_source` 标记，source 不再上报自动清除）

### 4.4 ReportStore 聚合逻辑（[查看](file:///workspace/gpm-web-admin/app/report_store.py)）— 核心

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

### 4.5 自动更新机制（[查看](file:///workspace/gpm-web-admin/app/updater.py)）

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

**更新路由**（[查看](file:///workspace/gpm-web-admin/app/routes/update.py)，需 token）：
- `GET /update/status`、`POST /update/check`、`POST /update/apply`、`PATCH /update/auto`

### 4.6 前端结构

**index.html + app.js**（仪表盘，[查看](file:///workspace/gpm-web-admin/static/app.js)）：
- `API = '/api/v1/dashboard'`，5 秒轮询
- 总体灯色区 + 摘要卡（在线数/整合包数/模组数）+ 灯色图例
- 三区块：服务端列表（`windows-server/web-server`）、客户端列表（`client`）、游戏推送条目
- 卡片左边框按 `light.level` 上色

**admin.html + admin.js**（服务端管理，[查看](file:///workspace/gpm-web-admin/static/admin.html)）：
- 5 Tab：整合包管理 / 模组管理 / 用户管理 / 配置 / 系统更新
- **加载器下拉含 6 项**：`auto / vanilla / forge / neoforge / fabric / quilt`（确认含 neoforge）
- 上传用 XMLHttpRequest 实现 `upload.onprogress` 进度回调
- `setInterval(loadStatus, 10000)` + `setInterval(loadUpdateStatus, 30000)`

---

## 五、gpm-client（Windows 客户端）

### 5.1 定位

Windows 桌面客户端（PySide6），从服务端同步条目，下载整合包后**自动安装模组加载器**并启动游戏。

- GitHub：`https://github.com/yzgmc/gpm-client`

### 5.2 目录结构

```
gpm-client/
├── run.py
├── app/
│   ├── main.py             # 入口（登录校验 + 启动 MainWindow + start_reporter）
│   ├── config.py           # ClientConfig + installed.json
│   ├── api_client.py       # 服务端 HTTP 客户端
│   ├── sync_manager.py     # 同步 + 本地状态
│   ├── downloader.py       # 多线程分块下载 + sha256 校验
│   ├── installer.py        # 整合包解压 + 模组复制
│   ├── launcher.py         # 通过适配器启动游戏
│   ├── loader_installer.py # Fabric/Forge/NeoForge/Quilt 自动安装（重点）
│   ├── reporter.py         # 心跳上报
│   └── ui/
│       ├── main_window.py  # 主窗口（3 Tab + 下载/加载器安装流程）
│       ├── widgets.py      # DownloadProgressDialog + LoaderInstallDialog
│       └── login_dialog.py
├── data/                   # client_config.json / installed.json / .reporter_id（gitignored）
├── requirements.txt
└── .github/workflows/build-release.yml
```

### 5.3 入口与配置

`app/main.py`（[查看](file:///workspace/gpm-client/app/main.py)）：
1. `import gpm_common` 触发适配器注册
2. `ClientConfig.load()`
3. 若无 token/username → 弹 `LoginDialog`（默认 `http://127.0.0.1:8001`），成功后写入 config 并 save
4. **融合体兜底**：`admin_url` 留空时用 `server_url` 兜底，用户填一个地址即可同步 + 上报
5. `MainWindow(config).show()` + `start_reporter(config)`

`ClientConfig` 字段（[查看](file:///workspace/gpm-client/app/config.py)）：
`server_url` / `install_base_dir` / `java_path` / `jvm_args`（默认 `["-Xmx4G","-Xms1G"]`）/ `last_sync_at` / `admin_url` / `reporter_interval` / `client_name` / `reporter_id`（持久化）/ `username` / `token`

环境变量：`GPM_ADMIN_URL` / `GPM_REPORTER_INTERVAL` / `GPM_CLIENT_NAME`

### 5.4 核心模块

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
- `install_mod`：通过 `install_dir_hint` 定位 mods 目录，`shutil.copy2`

#### launcher.py（[查看](file:///workspace/gpm-client/app/launcher.py)）
- `GameAdapterRegistry.require(game).build_launch_command(...)` → `subprocess.Popen(cmd, cwd=install_dir)`

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

#### reporter.py（[查看](file:///workspace/gpm-client/app/reporter.py)）
- `_build_heartbeat`：`kind="client"`、`username`（后台按用户去重）、`light=GREEN`（在线即绿）
- `metrics`：`installed_modpacks`（仅 modpacks 条目）/ `installed_modpack_count` / `last_sync_at` / `platform` / `install_base_dir`

### 5.5 UI

#### main_window.py（[查看](file:/workspace/gpm-client/app/ui/main_window.py)）

**跨线程信号机制**（注释强调）：工作线程通过 `_sig_*` 信号投递到主线程（AutoConnection 跨线程自动 QueuedConnection，`QDialog.exec()` 嵌套事件循环能正确接收）：
- 下载：`_sig_progress(int,int)` / `_sig_status(str)` / `_sig_close_dialog(int)` / `_sig_statusbar(str,int)` / `_sig_fail(str,str)`
- 加载器：`_sig_loader_progress(str,str,int)` / `_sig_loader_done(str)` / `_sig_loader_failed(str)`

3 Tab：整合包页（列表 + 详情 + 下载/启动）/ 模组页（表格 + 内嵌下载按钮）/ 设置页。

**下载流程** `_start_download`：
1. 创建 `DownloadProgressDialog`，`canceled` 连 `_cancel_event.set()`
2. 启动 `_download_worker` daemon 线程，`exec()` 阻塞
3. **整合包下载成功后**：`if kind == "modpacks" and accepted: self._maybe_install_loader(item)`

**加载器安装流程** `_maybe_install_loader`（[查看](file:///workspace/gpm-client/app/ui/main_window.py)）：
- `loader == "vanilla"` → 直接 return
- `loader not in SUPPORTED_LOADERS` → `show_info` 提示手动安装
- 无 `game_version` → `show_info` 提示
- 创建 `LoaderInstallDialog`，启动 `_loader_worker` daemon 线程，`exec()` 阻塞

`_loader_worker`：调 `install_loader(...)`，成功 emit `_sig_loader_done`，取消/异常 emit `_sig_loader_failed`。

#### widgets.py（[查看](file:///workspace/gpm-client/app/ui/widgets.py)）

`LoaderInstallDialog` 多阶段进度对话框：
- `_STAGES = [("download", "下载安装器"), ("install", "安装加载器"), ("done", "完成")]`
- 每阶段一行：状态点 `QLabel` + 阶段名 + `QProgressBar`
- 状态点配色：当前 stage `●` 蓝 `#0A84FF`，已过 `●` 绿 `#30D158`，未到 `○` 灰 `#48484A`
- 日志区深色背景，保留最后 8 行
- `on_done`：所有阶段绿 100%，禁用取消，启用完成按钮
- `on_failed`：detail 红 `#FF453A`，取消按钮改"关闭"

#### login_dialog.py
`LoginDialog` 返回 `(server_url, username, token)`，回车链（用户名→密码→登录）。

### 5.6 加载器自动安装完整流程（重点）

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
   ↓ _on_launch → GameAdapterRegistry.require("minecraft")
   ↓ adapter.build_launch_command（按 loader 找 jar 生成命令）
   ↓ subprocess.Popen(cmd, cwd=install_dir)
```

### 5.7 构建发布

`.github/workflows/build-release.yml`：Nuitka 单文件 exe
- 触发：`workflow_dispatch` + `push tags: ['v*']`
- `windows-latest`，Python 3.11
- `--onefile --standalone --windows-console-mode=disable --enable-plugin=pyside6`
- `--include-package=gpm_common --include-package=app`
- 生成 `gpm-client.exe`，配置存 exe 同级 `data/`

---

## 六、跨仓库协作流程

### 6.1 整合包从上传到启动的完整链路

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
       → 下载安装器到 .cache/loaders
       → 解析最新 loader 版本
       → java -jar 安装到 install_dir

4. 用户点"启动"
   → adapter.build_launch_command（按 loader 找 jar）
   → subprocess.Popen 启动游戏
```

### 6.2 心跳上报与仪表盘

```
各端启动（admin_url 非空）
   → Reporter 后台线程（默认 10s 间隔，启动立即上报一次）
   → POST /api/v1/report（Heartbeat：状态 + modpacks/mods 列表 + light + admin_users）
   → web-admin ReportStore.record（按 reporter_id/username 去重）
   → sync_admin_users（合并管理员账号）

前端 index.html 每 5s GET /api/v1/dashboard
   → ReportStore.aggregate
   → 总体灯色 + 按 kind 分组 + 推送条目聚合
```

### 6.3 用户同步机制

- 服务端上报时 `metrics.admin_users` 携带管理员 hash 列表
- web-admin `sync_admin_users(admin_users, source=reporter_id)`：
  - 先清除该 source 之前同步的带 `_source` 用户
  - 本地用户（无 `_source`）hash 不同则更新
  - 远端用户写入 `{hash, role:"admin", _source:source}`
- 实现"一处建管理员，多处可登录"

### 6.4 mod_loader 自动识别链路（跨 3 仓库）

```
gpm-client 上传表单（mod_loader=auto）
   → gpm-server/web-admin 路由调 adapter.detect_metadata
   → gpm-common MinecraftAdapter 识别（CurseForge/Modrinth/裸包）
   → 返回 mod_loader（含 neoforge）
   → 写入 meta.json
```

---

## 七、开发约定

### 7.1 协议与模型
- 所有端共用 `gpm_common.models`，避免字段不一致
- `API_PREFIX="/api/v1"`、`API_VERSION="1.0.0"`，所有端必须一致
- 路由路径用 `gpm_common.route("/xxx")` 拼接

### 7.2 路径安全
- 文件操作一律走 `gpm_common.safe_join` 防穿越
- id / 文件名经 `_sanitize` / `_sanitize_filename` 清理

### 7.3 认证模型
- JWT（HS256），默认 24h 过期，secret 来自 `GPM_AUTH_SECRET`（未设则进程内随机）
- 读操作（sync/list/download/status/games/config 读）开放
- 写操作（上传/删除/PATCH/改密/用户管理/config 改）需 `Authorization: Bearer <token>`

### 7.4 线程通信（客户端）
- UI 用 Qt Signal/Slot + QueuedConnection
- 工作线程（`threading.Thread`）通过 `_sig_*` 信号投递到主线程
- `QDialog.exec()` 嵌套事件循环能正确接收 queued 事件
- 取消用 `threading.Event` 贯穿下载器与加载器安装器

### 7.5 存储
- 无数据库，基于文件系统 JSON 元数据
- 整合包/模组：`data/{kind}/{id}/{meta.json, <file>}`
- 用户：`data/users.json`；配置：`data/server.json`；上报 ID：`data/.reporter_id`

### 7.6 错误处理
- 业务异常用 `GamePushError(code, status_code, message, details)`
- 认证异常用 `AuthError(message, status_code)`
- 返回体统一 `{"error":{"code","message","details"}}`

---

## 八、扩展指南

### 8.1 新增游戏

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

### 8.2 新增模组加载器

1. `gpm-common/adapters/minecraft.py`：
   - `supported_mod_loaders()` 加入新 loader
   - `build_launch_command` 加分支 + `_find_xxx_jar`
   - `detect_metadata` / `detect_mod_metadata` 加识别规则
2. `gpm-client/app/loader_installer.py`：
   - 实现 `_install_xxx`
   - 加入 `_INSTALLERS` 字典和 `SUPPORTED_LOADERS`
3. 前端下拉框（`admin.html` / `admin.js` / GUI `ModpackMetaDialog`）加入选项

### 8.3 新增 API

1. `gpm-common/protocol.py` 加路由常量（可选）
2. 对应仓库 `app/routes/xxx.py` 实现，`router = APIRouter()`
3. `app/main.py` 注册 `app.include_router(xxx.router)`
4. 鉴权按需加 `Depends(require_token(settings.auth_secret))`

---

## 九、已知问题与落差

### 9.1 gpm-server
- `static/admin.html` 有"系统更新" Tab，`admin.js` 调用 `/api/v1/update/*` 四端点，但 `app/routes/` **无对应实现**（功能未实现，仅前端 UI 存在）
- `static/index.html` 引用 `/static/app.js` 但该文件不存在，且未被路由暴露（遗留页面，应为 web-admin 的仪表盘误同步过来）
- Web admin.html 加载器下拉**没有 neoforge 选项**（但 GUI `ModpackMetaDialog` 有，服务端存储支持 neoforge）—— UI 不一致点

### 9.2 gpm-web-admin
- 写操作挂了 `_require_auth` 但未额外校验"是否为 admin"，任何登录用户均可增删用户（靠 UI 行为约束）
- 自动更新依赖 systemd `Restart=always`，非 systemd 环境需手动重启

### 9.3 gpm-client
- 首次运行 Nuitka exe 稍慢（自解压临时文件）
- `admin_url` 为空时用 `server_url` 兜底（融合体设计），独立后台场景需手动配置

### 9.4 跨仓库
- `gpm-common` 模型 `mod_loader` 字段注释仅写 4 项，实际支持 5 项（含 neoforge）—— 文档与代码不一致
- 各仓库 `gpm-common` 依赖从 GitHub `yzgmc/gpm-common` 安装，需保持版本同步

---

## 十、给后续 AI 的开发指引

### 10.1 快速上手

1. **先读本文件**了解全局架构
2. 改动前先读对应文件（用 file:/// 链接定位）
3. 4 个仓库的依赖关系：`gpm-common` 是基础，改它会影响所有端
4. 提交规范：各子仓库独立提交，主仓库（本仓库）用于存放功能说明文档

### 10.2 仓库地址

| 仓库 | GitHub |
|------|--------|
| 主仓库（本文档所在） | `https://github.com/yzgmc/Game-Push-Manager` |
| gpm-common | `https://github.com/yzgmc/gpm-common` |
| gpm-server | `https://github.com/yzgmc/gpm-server` |
| gpm-web-admin | `https://github.com/yzgmc/gpm-web-admin` |
| gpm-client | `https://github.com/yzgmc/gpm-client` |

### 10.3 本地开发环境

```bash
# 1. 克隆 4 个仓库到同一父目录
git clone https://github.com/yzgmc/gpm-common
git clone https://github.com/yzgmc/gpm-server
git clone https://github.com/yzgmc/gpm-web-admin
git clone https://github.com/yzgmc/gpm-client

# 2. 安装共享库（开发模式）
cd gpm-common && pip install -e .

# 3. 各端安装依赖
cd ../gpm-web-admin && pip install -r requirements.txt
cd ../gpm-server && pip install -r requirements.txt
cd ../gpm-client && pip install -r requirements.txt

# 4. 运行（融合体最常用，单进程即可调试全部流程）
cd gpm-web-admin && python run.py    # http://localhost:8001
# 或服务端
cd gpm-server && python run_gui.py   # GUI + 后台服务
# 或客户端
cd gpm-client && python run.py       # 需先启动服务端/融合体
```

### 10.4 调试技巧

- **融合体自上报**：启动 gpm-web-admin 后，仪表盘会自动出现一个 `web-server` 上报端（自己上报给自己）
- **默认账号**：`admin / admin123`
- **数据目录**：源码运行在仓库根 `data/`，打包后在 exe 同级 `data/`
- **日志**：`gpm-deploy.sh` 部署的用 `journalctl -u gpm-web-admin -f`
- **协议版本不一致**会报 `PROTOCOL_MISMATCH`，确保各端 `gpm-common` 版本同步

### 10.5 常见任务速查

| 任务 | 改哪里 |
|------|--------|
| 加新游戏 | `gpm-common/gpm_common/adapters/` 新建 + 注册 |
| 加新加载器 | `gpm-common/adapters/minecraft.py` + `gpm-client/loader_installer.py` + 前端下拉 |
| 加新 API | 对应仓库 `app/routes/` + `app/main.py` 注册 |
| 改数据模型 | `gpm-common/gpm_common/models.py`（注意向后兼容） |
| 改灯色阈值 | 环境变量 `GPM_LIGHT_*` 或 `gpm-common/lights.py` |
| 改认证 | `gpm-common/auth.py` |
| 改仪表盘聚合 | `gpm-web-admin/app/report_store.py` 的 `aggregate()` |
| 改加载器安装 | `gpm-client/app/loader_installer.py` |
| 改自动更新 | `gpm-web-admin/app/updater.py` |

### 10.6 提交与发布

- 各子仓库独立 git 提交，建议 conventional commit
- 打 tag 触发 GitHub Actions 自动构建 exe（`v*` tag）
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

### gpm-server
- [app/main.py](file:///workspace/gpm-server/app/main.py)
- [app/config.py](file:///workspace/gpm-server/app/config.py)
- [app/storage.py](file:///workspace/gpm-server/app/storage.py)
- [app/reporter.py](file:///workspace/gpm-server/app/reporter.py)
- [app/gui.py](file:///workspace/gpm-server/app/gui.py)
- [app/routes/modpacks.py](file:///workspace/gpm-server/app/routes/modpacks.py)
- [app/routes/auth.py](file:///workspace/gpm-server/app/routes/auth.py)
- [static/admin.html](file:///workspace/gpm-server/static/admin.html)

### gpm-web-admin
- [app/main.py](file:///workspace/gpm-web-admin/app/main.py)
- [app/config.py](file:///workspace/gpm-web-admin/app/config.py)
- [app/report_store.py](file:///workspace/gpm-web-admin/app/report_store.py)
- [app/updater.py](file:///workspace/gpm-web-admin/app/updater.py)
- [app/reporter.py](file:///workspace/gpm-web-admin/app/reporter.py)
- [app/routes/report.py](file:///workspace/gpm-web-admin/app/routes/report.py)
- [app/routes/dashboard.py](file:///workspace/gpm-web-admin/app/routes/dashboard.py)
- [app/routes/update.py](file:///workspace/gpm-web-admin/app/routes/update.py)
- [static/admin.html](file:///workspace/gpm-web-admin/static/admin.html)
- [static/app.js](file:///workspace/gpm-web-admin/static/app.js)

### gpm-client
- [app/main.py](file:///workspace/gpm-client/app/main.py)
- [app/config.py](file:///workspace/gpm-client/app/config.py)
- [app/loader_installer.py](file:///workspace/gpm-client/app/loader_installer.py)
- [app/downloader.py](file:///workspace/gpm-client/app/downloader.py)
- [app/installer.py](file:///workspace/gpm-client/app/installer.py)
- [app/launcher.py](file:///workspace/gpm-client/app/launcher.py)
- [app/reporter.py](file:///workspace/gpm-client/app/reporter.py)
- [app/ui/main_window.py](file:///workspace/gpm-client/app/ui/main_window.py)
- [app/ui/widgets.py](file:///workspace/gpm-client/app/ui/widgets.py)
- [app/ui/login_dialog.py](file:///workspace/gpm-client/app/ui/login_dialog.py)

---

*本文档由深度研究 4 个仓库源码后编写，覆盖全部模块、路由、关键函数与协作流程。后续 AI 仅需阅读本文件即可继续开发，无需重新探索。*
