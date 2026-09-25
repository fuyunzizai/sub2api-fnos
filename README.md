# Sub2API 飞牛安装包

这个目录用于把 [Wei-Shaw/sub2api](https://github.com/Wei-Shaw/sub2api) 打包成飞牛 FPK 应用。

安装后只会启动一个 Docker Compose 项目，并且只运行一个容器：

- 容器名：`sub2api`
- 飞牛包版本：`0.2.8`
- 基础镜像名：`sub2api-fnos:0.2.8`
- Sub2API 运行二进制：安装/升级时从官方 Release 下载 `0.2.8` 到应用数据目录，后台“立即更新”会继续更新这个持久化二进制。
- 网络模式：host
- 访问端口：`0.0.0.0:8088`

PostgreSQL 和 Redis 已经内嵌到同一个容器里，不会再额外启动数据库或 Redis 容器。所有应用数据、数据库数据、Redis 数据和日志都保存在飞牛所选安装盘的 `@appshare/sub2api-docker/data` 目录中。

安装包会优先拉取 GitHub 容器仓库里的预构建镜像 `ghcr.io/fuyunzizai/sub2api-fnos:0.2.8`（基础环境镜像，二进制由安装脚本另行下载到 data/runtime），避免在 NAS 上本地编译。`PREBUILT_IMAGE` 支持空格分隔的多个候选 tag，会依次尝试，全部失败才回退到用包内 `Dockerfile` 本地构建（需要 NAS 能访问 DockerHub 与 Alpine 软件源）。

Windows 本地打包：

```powershell
.\build_sub2api_fpk.ps1
```

打包为拉取预构建镜像的 FPK：

```powershell
.\build_sub2api_fpk.ps1 -Image "ghcr.io/fuyunzizai/sub2api-fnos:0.2.8"
```

生成的安装包会写入 `dist/sub2api-docker_0.2.8.fpk`。

GitHub Actions：

- 推送这个目录到 GitHub 仓库。
- 在 Actions 页面运行 `Build Sub2API fnOS Image`。
- 工作流用于构建基础镜像。当前 FPK 使用已发布的 `ghcr.io/fuyunzizai/sub2api-fnos:0.2.8`，运行程序也会由安装脚本从 Sub2API 官方 Release 下载到持久化数据目录。
- 镜像发布后，用 `-Image` 重新打包 FPK，飞牛安装时就会直接拉预构建镜像。

安装说明：

- Sub2API 监听 `0.0.0.0:8088`。
- PostgreSQL 监听容器内 `127.0.0.1:15432`。
- Redis 监听容器内 `127.0.0.1:16379`。
- 安装向导填写的管理员邮箱和密码会在数据库初始化后强制同步，确保可以直接登录。
- 数据库、Redis、JWT 和 TOTP 密钥会自动生成并保存到 `sub2api.env`。
- 卸载会删除容器、镜像、应用数据、日志、Docker 资源和旧版本可能创建的系统 PostgreSQL 数据库账号。

安装失败排查：

- 飞牛只提示“执行脚本出错且原因未知”，真实原因在安装日志里：`/tmp/sub2api-install.log`，成功/失败都会再复制一份到 `@appshare/sub2api-docker/install.log`。
- 直接手动复现（SSH）：
  ```bash
  CB=$(ls /vol*/@appmeta/sub2api/cmd/install_callback /vol*/@appcenter/sub2api/cmd/install_callback 2>/dev/null | head -1)
  bash -x "$CB" 2>&1 | tail -80
  ```
- 常见原因：GitHub Release 下载超时（已改为非致命，容器会用镜像内二进制）、GHCR/DockerHub 不可达、`docker compose` 插件缺失（已兼容 `docker-compose`）、8088 端口被占用。

0.2.8 兼容性说明：

- 上游 0.2.x 改为 `config.yaml` + 环境变量（viper）双重配置，`config.yaml` 不是必需的，缺失时回退默认值。
- 本包仍通过 `docker-compose.yaml` 环境变量注入数据库、Redis、JWT、TOTP 与管理员账号，入口脚本另外把同值写入 `/app/data/config.yaml`（`/app/data` 在上游配置搜索路径内）。环境变量优先级高于该文件。
- 已确认 `DATABASE_HOST/USER/DBNAME/SSLMODE`、`REDIS_HOST/PASSWORD`、`SERVER_HOST/PORT`、`RUN_MODE`、`SETUP_MIGRATION_TIMEOUT_SECONDS`、`AUTO_SETUP` 在 0.2.8 仍生效。
- `SUB2API_WRITE_CONFIG` 在上游 0.2.8 已移除，仅作为本包入口脚本自身的开关保留，不影响运行。
- 时区：上游 0.2.8 只认 `TZ` 环境变量，旧版 config.yaml 的 `timezone` 键已废弃（compose 已设置 `TZ=Asia/Shanghai`）。

